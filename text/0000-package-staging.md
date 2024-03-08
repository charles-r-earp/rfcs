- Feature Name: `package_staging`
- Start Date: 2024-03-07
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
Create a package staging area. Packages can be published to staging. Staged packages can be published to the index, replaced, or deleted. 

# Motivation
Package names are often reserved by uploading an empty crate. In many cases the author abandons the project. A fledgling rustacean might want some additional feedback on their project, before committing to it. Staging allows reserving a package name, and verifying a package, without permanently adding it to the index.

[Packaging and publishing multiple crates](https://github.com/rust-lang/cargo/issues/1169) currently requires packaging and publishing packages in order. While there are some community maintained tools that help automate publishing workspaces, they can't ensure that errors won't occur, and when they do, it leaves the workspace in an indeterminate state, with some packages published and others unpublished. Typically it is useful to tag released commits, but if some packages can't be published without fixes, this is no longer possible. Package staging would allow cargo and crates-io to validate all packages, before adding them to the index. 

It may be useful for crates-io to quarantine crates, to prevent them from entering the index. This could be particularly disruptive when publishing many packages that depend on each other. Staging affected packages would allow the author to publish after the quarantined packages are validated. 

Organizations may want to require multiple sign offs before publishing packages. Packages could also be staged by automation, particularly helpful with many large uploads. Staging could enable or otherwise improve these options.

# Guide-level explanation
Add a package staging area to crates-io. The API will support publishing packages to staging, releasing packages to the index, deleting staged packages, and searching for staged packages.

- `cargo package --stage` allows packaging multiple packages that depend on each other. 
- `cargo publish --stage` uploads packages to staging, without releasing them. 
- `cargo publish --release` releases staged packages. Packages specified with `-p` or `--package` do not have to be local to the workspace.
- `cargo publish --stage --release` will release packages on successfully uploading them.
- `cargo unpublish` deletes packages from staging. 
- `cargo search --staged` lists packages the user owns.

# Reference-level explanation
The additional functionality is added to nightly cargo, gated with `-Z unstable-options`.

This RFC describes motivating features "packaging and publishing multiple crates at once" and "crate quarantine", but may be minimally implemented without them. Accepting this RFC does not require accepting these features. 

Currently, packaging multiple packages that depend on each other will fail. However, multiple packages can be packaged if they do not depend on each other. Packaging of multiple crates may require significant changes, including constructing a DAG of crate dependencies, and determining a new order to package them in, as well as a `[patch.crates-io]` table. It is therefore easier to make this functionality opt in. Potentially the `--stage` flag could be removed on stabilization, or no longer required with an edition change. 

Publishing multiple packages can be done via `cargo publish --stage --release`. The packages will be packaged together, then uploaded one at a time in order. When all packages are staged successfully, will release all the packages. 

Staged packages are not added to the index, and cannot be downloaded.

In the case of quarantine, packages that cannot be published will be staged. When / if the quarantine is lifted, the owner could be notified to release the packages. 

Only one package version of a package can be staged. When a package is staged, it will overwrite the previous one. 

When a new package is staged, the author will gain ownership of the package. If a package is unpublished, and does not have a released version, the author will no longer own the package, and it can be used by another author. 

crates-io usage policy will apply to staged packages. crates-io may accept, reject, or quarantine a package when it is uploaded. 

Add a `stage` endpoint with authorization, supports `cargo publish --stage`.

Add a `release` endpoint with authorization. When a package is released from staging, it's dependencies must have a released version. Ensures all packages can be released, in the provided order. 

When cargo releases multiple packages, cargo should ensure that they are staged by the owner. Cargo will determine and verify the order that packages can be released in, and release them, either one at a time, or all together. 

Add a `delete` endpoint with authorization. Staged packages can be deleted. If they are quarantined, the package will be retained for investigation.

Add `staged: bool` argument to `search` endpoint. Will search staged packages owned by the user. Requires authorization.

When stabilized, crates-io website should support similar functionality. Staged packages are listed, and can be published or deleted. May also show warnings / errors, like missing manifest keys, or when a package is quarantined. 

# Drawbacks
This RFC proposes several new cargo command line options and web API changes, and potentially requires some coordination between teams. 

The new functionality may require significant refactoring to mitigate logic duplication, in the case of staged versus not staged code paths, both in cargo and crates-io. 

Package staging makes squatting of package names somewhat easier, and prevents the community from policing such crates, as they wouldn't be able to see what was uploaded, or any information other than who owns the package. 

This RFC is motivated by several features, including "packaging / publishing multiple crates at once" and "crate quarantine". It is less useful without at least one of those features.

# Rationale and alternatives
Cargo and crates-io could have a more explicit way to publish workspaces and / or multiple packages at once. This might be simpler and provide a better user experience. However, because crate uploads are limited to 10 mb to mitigate Denial-of-Service attacks, it might be difficult to make publishing "atomic". With package staging, packages can be staged and released independently, and it's not ambiguous what should happen on publish failure. 

The exact API could be altered, options / endpoints renamed. For example, could adopt npm's `publish --access public|restricted`. Staging was chosen because it best describes the intent, which is to break publishing into stage and release steps, rather than allow users to use crates-io as a private registry. 

This RFC does not support downloading staged packages. This could be useful for validation or for others to review. Potentially users could test installing or using their packages. But this complicates versioning, as package versions are not unique, nor are staged packages permanent. Better to not have cargo deal with lockfiles containing packages from staging. 

Publishing packages yanked was also considered. This could reuse existing means to unyank packages, which is similar to `publish --release`. However, it's less clear that a package that is published yanked can be deleted / altered, unlike a package that was published normally and then yanked. Yanking has a specific purpose, which is different from staging.

# Prior art
- [RFC: Add staging workflow for CI and human interoperability #92](https://github.com/npm/rfcs/pull/92)
One inspiration for this RFC. With npm, the `promote` command would be equivalent to `publish --release`. A separate command might have better clarity, but requires duplicating many additional options, and doesn't allow chaining `--stage` and `--release`, to facilitate publishing multiple packages. `publish` is a permanent (now only potentially) action that users are already aware of, so it could be confusing to introduce another one that has similar effect.  

- [RFC: Nested Cargo packages #3452](https://github.com/rust-lang/rfcs/pull/3452)
Nested packages are a neat idea, but don't cover many workspace configurations. It assumes that the parent package will have many nested / private dependencies. For example, [wgpu](https://github.com/gfx-rs/wgpu) is a graphics API that includes naga, a shader translation API, which could be used independently. So nested packages would not cover the broader ecosystem, and are not a true replacement for publishing multiple packages in a workspace. 
Due to the 10 mb size limit mentioned previously, nested packages could potentially be better implemented via staging. 

- [Crate quarantine](https://github.com/rust-lang/rfcs/pull/3464)
A motivation for this RFC. See this [comment](https://github.com/rust-lang/rfcs/pull/3464#issuecomment-1667764606) "Under this proposal, quarantining a crate could very seriously complicate publication of a multi-crate workspace (or other kind of "deep stack"). Rust wants us to make smaller crates for separate compilation reasons, but cargo then insists that these crates which can be conceptually part of a single program must be published separately. For release management to be practical, there must be a way to publish a workspace even if some of the involved crates end up quarantined." (snipped). Staging allows packages to be quarantined on upload, packages can be released together, such that if any package is quarantined, the operation fails.

# Unresolved questions
The exact API, names of command line options, endpoints, or other details could be changed. The RFC could be implemented, and the names changed prior to stabilization. 

# Future possibilities
This is more or less covered in the motivation section. 


