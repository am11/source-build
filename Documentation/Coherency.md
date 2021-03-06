# Problem
Official non-servicing builds of the .NET Core SDK are *coherent*, meaning that for each repo that contributes bits to the final product, exactly one version of that repo is used at all stages of the build.  For instance, CLI and core-setup both depend on CoreFX.  In a coherent build, this will be the same version of CoreFX, but in an incoherent build, CLI could have been built with a different version than core-setup.  This can cause problems which usually manifest in source-build as build errors along the lines of `Could not load file or assembly 'NuGet.Frameworks, Version=4.9.3.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35'. The located assembly's manifest definition does not match the assembly reference.`, but could also result in runtime errors.  

Source-build has traditionally worked around this by forcing all repos to use the same bits (the source-built ones) throughout the build process.  However, this brings in new problems:
- We are not necessarily shipping the exact bits that an official release would.  In some cases, our bits can actually be invalid in ways we won't see.
- Servicing releases *should* be incoherent.  We do not ship non-serviced packages, so a valid official SDK could have some 3.0.1 bits and some 3.0.0 bits.
	- This also means that we do not get all the packages we need from our forced-coherent build.
- Very few official builds are coherent during development.  Builds diverge from coherence within days after release and soon become massively incoherent (dependency graph reports 44 distinct repo-shas have packages depended on by today's CoreFX build, and 181 distinct repo-shas dependencies for core-sdk).  We would like to support daily builds, which means that we have to deal with incoherence during the development process.

# Meeting Notes 2019-05-03/Open Questions
- Some incoherent builds are buildable and run fine and some do not - there's really no way we can tell until we try to build the entire product and it either works or doesn't.  @mmitche has been thinking about ranking incoherencies.
	- Coherency might not be the most important thing.
		- Useful for prebuilts, knowing whether a particular repo change broke source-build
- Consumption and production might have to be done at the same time - merging of phases 2 and 3 below.
	- Or we could keep `PreferredHashes.props` and only do coherent builds
- What value do we get from the plan?
	- Only doing coherent builds makes it hard for repo owners to fix and try out things.
	- Could help with driving down prebuilts.
	- Coherent builds might be going to 2/release.
	- Repo isolation helps us out with a few different problems we have - isolating prebuilts, incremental builds, repos that need reference-only packages vs. ones that need the full package.
- Could do parallel builds with ProdCon
	- Instead of feeding in MyGet etc packages, feed in source-built packages, then tar up with prebuilts for that repo only and build offline
	- Phase 3-4 -> integration with ProdCon?
- We should subscribe to all child repos but building from master would uberclone from core-sdk hash and use those hashes
- CI legs per repo - can do this in source-build uptake PRs.
	- Could we put the infrastructure for this in Arcade? (or have them do it)
- Early warning CI (in the client repos) - have to have separate build definitions per repo.
- Warnings for repo uptake "currency" - @mmitche and @dleeapho talked about this - if something breaks we would get notified?
	- Can currently sit for a while until someone realizes it's not being uptook.
- How do we handle breaking changes?
	- If we patch it gets messy - possibly have to have the same patch rebased on many different hashes.
	- If we don't patch, we end up with holes in what source-bulid can build that propagate to all consuming builds as well.
- Servicing - we can do multiple hashes per repo without doing the entire coherency plan.
	- Make duplicate `repo.proj` files and multiple copies of a submodule.
	- Would mean we don't have to do the entire graph of dependencies, just the ones that actually produce things we want to ship.
	- Costing?  Infrastructure is already all there, should be mostly pain-free.
- We can't currently ship nupkgs so servicing is hard for source build - need RPM/debs of nupkgs or similar?
- Is this only a CoreFX problem?  Possibly, if other repos don't want to service packages it doesn't apply.
	- CoreFX packages are "platform extension" packages.
	- This could be a valid limitation of source-build - platform extensions are not source-built and are grabbed from NuGet.
		- Only needed if you're referencing them
	- "Platform" == NETCoreApp, platform extensions are extra.
- @dseefeld to set up meeting with the CoreFX team and @mmitche to discuss.

# Solutions Considered
We have considered a few solutions:

## Continue with traditional source-build coherence
This would mean that we would continue to force coherence on every build, using our traditional method of passing around PackageVersions.props as the build progresses.  We would continue to use only one commit per repo, usually the latest on the relevant branch (e.g. release/3.0).  Currently we have to sometimes manually change commits if not all the repos' latest commits work together but if we're getting updates from BAR this problem should mostly go away.

Pros:
- Simple.
- Already have this mechanism.

Cons:
- We continue to not ship bits that match official builds, except on releases.
- We have to deal with the problem of not getting all packages for servicing releases:
	-  Issues https://github.com/dotnet/source-build/issues/210 and https://github.com/dotnet/source-build/issues/923 are our earlier discussions.
	-  If we turn on BuildAllConfigurations and BuildAllPackages, we will ship bits that do not exist except on RHEL machines.  If someone shares a project but not the packages directory, restoring these packages that do not officially exist will fail.
	-  In the worst case, these packages that we force to build could be known-bad code (that is okay normally because that package is not shipping).
	-  If we don't turn on BuildAllConfigurations and BuildAllPackages, we will continue to have a large number of prebuilt packages.

## Build multiple versions of each repo
This would mean we build CLI with its expected version of CoreFX, and core-setup with its own expected version of CoreFX.

Pros:
- We match the official bits very closely.
- We don't have to deal with servicing considerations.

Cons:
- This explodes rather quickly and would have extremely long build times.
- We have to be careful about cross-contaminating package feeds during the builds.

## Store and download packages in later builds
During each build, we would save source-built packages off in an Azure account or NuGet feed.  During later builds, we would restore these packages and use them in builds later in the dependency tree.  We can store the packages in multiple formats; nupkgs for us or RPM/Debs to allow distro maintainers to save the packages and reuse them.

Pros:
- Improved build times.
	- Fringe benefit: incremental builds would probably finally work.
- Still lets us eliminate prebuilt packages (all packages stored are source-built).
- We match the official bits very closely.

Cons:
- Have to be very careful to not contaminate the storage with prebuilt packages.
- Need internet access during the production dev and daily builds.
- Need to carry required packages in a tarball or to an N+1 build (could be nupkg, tar.gz, RPM, deb).

# Proposed Solution
Hybrid approach: we will enable storing and retrieving packages, but allow building multiple versions of each repo completely from source.  Daily builds and dev builds will mostly use the stored packages approach while release builds will use the completely-built-from-source approach.  Doing multiple versions of repos only for release builds should cut down on the numbers of builds that are done many times since these builds should be mostly coherent except for servicing considerations.

## Phase 1 (early week of 5/6)
After Darc merges uberclone support (https://github.com/dotnet/arcade-services/pull/291), update our version of Darc and get rid of submodules, using an uberclone instead.  This will give us many more repos than we will actually use.  For the time being, use some `PreferredRepoHashes.props` or similar to tell the build which one to use - this should change nothing in our output.  We may also be able to use a heuristic, such as the latest commit of each repo, to automate this process.  At this point we should be able to take daily build updates but will be producing a product that differs from the official release, and will have problems if we don't implement later phases before servicing.

## Phase 2 (1 sprint)
Implement some parts of repo isolation (https://github.com/dotnet/source-build/issues/928).    Next implement publishing these feeds to our package storage.  It's nice to do this even if we don't have consumption yet because we will build up a cache of source-built packages we can use later.

## Phase 3
(1 sprint)
Use a separate output feed for each repo, then supply only the feeds that repo.projs say they depend on to each repo build.

(2 sprints, blocked by patch solution/decision)
Change to using each repo's `Version.Details.xml` to determine its inputs.  This includes the hash and the exact packages and version numbers of the repos, so after we have this we can get rid of `PreferredRepoHashes.props` and flip the switch to allow multiple builds per repo.  We will also be building the same bits as the official release.  We will also have to be very far along on patch removal at this point, or we'll run into lots of problems trying to keep multiple versions of patches straight.

## Phase 4
Allow consuming packages from the package storage.  This can either be a master switch or a per-repo determination to allow easier incremental builds.

# Appendix
## Storage format proposal
- Use AzDo build storage
- Use a source-build BAR channel
- Channel runs a release pipeline to upload to internal or external storage as appropriate
- Build numbers are significant to us so we will have to hold on to a manifest or similar with those.