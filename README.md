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
6. Team members will comment on your RFC PR and hold an RFC meeting about that.
7. If your RFC is accepted, the team member will add a label `status/accepted`.
8. Create a tracking issue and start your implementation.
9. During your implementation, minor changes to the RFC are allowed.
10. After your implementation is merged into [RisingWave][risingwave], you can merge the RFC.

## The RFC life-cycle

The life cycle of an RFC is simple enough.

```plain
Active ----> Accepted ----> Completed
        |
        |--> Closed
```

In every step, we can use the PR to discuss the RFC; even if it's closed or completed.

## Contributing

Most changes to the repo are the introduction of new RFCs, but some improvements to the repo itself are welcome, such as adding some lints to our CI.

**note**
Such PRs should not be combined with changes to any specified RFCs.

[risingwave]: https://github.com/risingwavelabs/risingwave
