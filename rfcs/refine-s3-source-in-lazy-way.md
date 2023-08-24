---
feature: Refine S3 Source List Objects in A Lazy Way
authors:
- "TabVersion"
start_date: "2023/08/23"
---

# Refine S3 Source: List objects in A Lazy Way

## Background

In the current implementation, the S3 Source lists all files during initialization, 
which can lead to prolonged initialization times when there are a large number of files. 
Specifically, the S3 API limits the return to a maximum of 1000 files per request, 
and we make multiple requests until pagination is complete. 
In scenarios with too many files (more than 1 million), the time taken by `list_objects`
exceeds 10 seconds, causing the creation of `source` to timeout and the DDL fails.

Also, in the current implementation, the `Source Manager` completes the mapping 
from filenames to `Source Executor` when listing all files and needs to persist the mapping relationship. 
When new files are added (`ChangeSplit`) or the cluster recovers (`Add`), 
the barrier needs to pass the full mapping relationship to each Source Executor. 
This could obviously result in excessive transmission overhead and prolong the recovery time.

## Solution

Overall, the main idea is to control the number of ongoing files to reduce the barrier 
payload size and the time taken by `list_objects`.

### Collect Finished Files

In the current implementation, the `Source Manager` maintains all the S3 objects and persists the mapping relationship of 
filenames to the `source executor` in the table fragment. It's evident that during cluster restarts, `Source Manager` 
needs to transmit the entire mapping relationship to every `Source Executor`. This can lead to excessive transmission overhead, 
prolonging the recovery time and potentially stalling the entire recovery process. 
As time progresses, the number of files that have been read increases. However, we still need to send the entire mapping 
relationship every time, which means that most of the transmitted data pertains to files that have already been read, 
rather than files that need to be read.

An obvious optimization strategy is to avoid sending files that have already been read. We can maintain a list of 
finished files in `Source Manager` and clear the related status in the table fragment. 
When transmitting the mapping relationship, only the unfinished files should be sent.

To achieve this goal, a proper mechanism is needed to recycle the list of finished files. 
The good news is that the barrier can assist in this process. When each checkpoint arrives, the source executor can 
write the filenames of the finished files from the state cache into the barrier. 
Eventually, through barrier collection, this information is transmitted to the Source Manager, completing the process. 
The changes involved here include adding a new field or Mutation called `FinishedFiles` in the barrier 
and introducing a new state `finishedFiles` in `Source Manager`. 
This status will record the filenames of the finished or deleted files and needs to be persisted.
Additionally, the `collect_barrier` needs to be modified to support retrieving `FinishedFiles` from the finished barriers.

Another optimization is that since each barrier will carry at most one mutation, the content of Add/SplitChange 
can be extracted when flowing through the `source executor`, reducing the overhead of subsequent transmissions.

### Control the Number of Ongoing Files

In the previously mentioned solution, we focused on how to avoid sending too many files during the reading process. 
However, if we send all the files in the s3 bucket at once, we still face issues with an overly large barrier and prolonged list objects times. 
Therefore, we can attempt to control the number of files sent each time, allowing us to manage the overall load on the cluster.

From our introduction to the S3 Source, we know that the Source Manager can determine the total number of files 
and the number of files that have been completed. With this information, we can ascertain the number of files currently being read, aka. ongoing files. 
Taking into account the limitations of the S3 API, I propose a solution based on reading file list pagination 
according to the number of files being read.

1. Define the current page `cur_page`, initialized to 1 (indicating the max page per `list objects` request), 
and the trigger to read the next page's file count `MIN_ONGOING_FILE_NUM`. This value should be less than 1000.

2. During initialization, read all the files on the first page and add them to the reading queue. Assuming there's pagination,
the number of ongoing files would be 1000.

3. As the reading progresses, the number of files being read will gradually decrease. 
When the number of ongoing files drops below `MIN_ONGOING_FILE_NUM`, allow the next page of the file list to be read; i.e., `cur_page += 1`.

4. When the next enumerator tick arrives, read the file lists of pages `[0, cur_page)` and add the unread files in the pages to the reading queue.

During this process, when the number of files being read exceeds `MIN_ONGOING_FILE_NUM`, 
even if the request results on the first page change, the list of files being read remains unchanged. 
The S3 API ensures that the listed file list is ordered, so we won't get significantly different results when listing objects consecutively.

### (Optional) Pass Split Changes in Delta

In the solutions mentioned above, we always transmit the complete file list in both `AddMutation` and `SplitChangeMutation`. 
However, within `SplitChangeMutation`, we can opt to send only the newly added file list, which can reduce the amount of data transferred.

I believe this is a minor optimization, but its impact might not be significant since `SplitChangeMutation` don't occur very frequently.
