---
feature: S3 source with csv parser
authors:
  - "Mingchao Wu"
start_date: "2022/11/23"
---

# My Excited Feature

## Summary

Support load data from Amazon S3 and other Simple Storage Services with CSV format.


## Design

To implement the S3 data source, an abstraction of the file system data source is proposed.

### Overall Process

`Split Enumerator`: Continuously list all files/objects in the directory/bucket

`FsSourceExecutor`:

1. get all splits from barrier mutation
2. scan info of all finished splits from the state store, then filter out finished splits
3. start reading the filtered splits
4. when taking a snapshot, a special prefix is used to identify the finished split.

### Implementation

- Split

  ```rust
  pub struct FsSplit {
      pub name: String,
      pub offset: usize,
      pub size: usize,
  }
  ```
  The `S3SplitEnumerator` will list all objects in a bucket. Now, the interval between two invocations to `list_splits` is 10s, is this value suitable for fs source?

- Split reader

  A `FsSplitReader` trait. The reader will reader all  `splits` passed to it and keeps yielding `FsSourceMessage`. The `FsSourceMessage` contains the `split_size`  to remind source if a split has been finished (Another possible approach is to yield a message with an empty payload after each object has finished).

  ```rust
  #[async_trait]
  pub trait FsSplitReader: Sized {
      type Properties;
  
      async fn new(properties: Self::Properties, splits: Vec<FsSplit>) -> Result<Self>;
  
      fn into_stream(self) -> BoxFsSourceStream;
  }
  
  pub struct FsSourceMessage {
      pub payload: Option<Bytes>,
      pub offset: usize,
      pub split_size: usize,
      pub split_id: SplitId,
  }
  ```

- Parser

  A file is different from a message queue, where a file is read as a series of byte chunks, instead of separate messages.

  The parser should try to parse one record from the payload, if the payload is not enough to parse one record, the parser will buffer the payload in its internal buffer and wait for more bytes to be passed. If any error occurs, the parser should clean its internal buffer.

  ```rust
  pub trait ByteStreamSourceParser: Send + Debug + 'static {
      type ParseResult<'a>: ParseFuture<'a, Result<Option<WriteGuard>>>;
      /// Parse the payload and append the result to the [`StreamChunk`] directly.
      /// If the payload is not enough to parse on record, the parser will buffer
      /// the payload and wait for more bytes to be passed.
      ///
      /// # Arguments
      ///
      /// - `self`: A needs to be a member method because some format like Protobuf needs to be
      ///   pre-compiled.
      /// - writer: Write exactly one record during a `parse` call.
      ///
      /// # Returns
      ///
      /// A [`WriteGuard`] to ensure that at least one record was appended or error occurred
      fn parse<'a, 'b, 'c>(
          &'a mut self,
          payload: &'a mut &'b [u8],
          writer: SourceStreamChunkRowWriter<'c>,
      ) -> Self::ParseResult<'a>
      where
          'b: 'a,
          'c: 'a;
  
  		// reset the state of the parser to parser a new split
  		// e.g. The csv parser needs to skip the header of the next file.
      fn reset_for_new_split(&mut self);
  }
  ```

## Alternatives

At which step should we split the data of the file into separate messages, parser or reader?

Another option is to use the delimiter to split the data into multiple separate messages when the `FsSplitReader` reads the file. This will make the implementation of Parser much simpler.

## Future possibilities

* Make the implementation of `FsSourceExecutor` more graceful.
