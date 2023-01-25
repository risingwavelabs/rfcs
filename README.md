# RisingWave RFCs

RFC (request for comments) for changes to [RisingWave][risingwave].

## When to create a new RFC?

Most changes can be made directly on GitHub pull request workflow, including minor features, fixes, and improvements.

You only need to follow the RFC process if:

* You want to refine the framework of some components.
* You need to implement a non-trivial feature.
* You want to change or introduce some core terminology.

## How to create a new RFC?

Following the process:

1. Create a new branch or fork the repo.
2. Copy 0000-template.md to rfcs/0000-my-feature.md. Don't assign an RFC number yet.
3. Fill in the RFC. Most template sections are optional, but don't forget to update the info in the front matter.
4. Submit a pull request to the repo.
5. Use the pull request number as your RFC number, then update the filename prefix.
6. Team members will comment on your RFC PR and hold an RFC meeting about that. After the meeting, a member should post a meeting summary on the PR.
7. If a consensus is reached among the team, a member should leave a comment to clarify the status of the RFC as accepted and add a label `status/accepted`. Remaining unresolved but not planned issues should also be summarized, if there's any.
8. Create a tracking issue and start your implementation.
9. It is allowed if the implementation contains minor differences from the RFC. The RFC should be updated correspondingly.
10. After your implementation is merged into [RisingWave][risingwave], you can merge the RFC.

## The RFC life-cycle

The life cycle of an RFC is simple enough.

```plain
Active ─┬──► Accepted ────► Completed
        │
        └──► Closed
```

In every step, we can use the PR to discuss the RFC; even if it's closed or completed.

## Contributing

Most changes to the repo are the introduction of new RFCs, but some improvements to the repo itself are welcome, such as adding some lints to our CI.

> **Note**
> Such PRs should not be combined with changes to any specified RFCs.

[risingwave]: https://github.com/risingwavelabs/risingwave
