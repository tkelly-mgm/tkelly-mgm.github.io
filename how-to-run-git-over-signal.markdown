# How to run Git over Signal

So, your organization’s SSO system has been breached and you don’t know who or what to trust and your management has mandated that the only permitted method of communicating is Signal. Well, you’re in luck. With the magic of packfiles and public key cryptography, you too can perform publish code and do code reviews over Signal, or any other trusted service that lets you send files and communicate with video.

## Everyone: Signatures

Since this is a low trust environment, you should first set up your local Git to sign your commits.

This requires Git 2.34.0 or higher.

If you don’t have signatures set up, proceed to “Adding your SSH private key to Git”. If you already have GPG signatures set up, continue to “Exporting your GPG key”.

### Adding your SSH private key to Git

1.  Set the signature method to SSH:
    
        git config --global gpg.format ssh

2.  Set your signing key to your SSH private key. Usually, your SSH key will be at `~/.ssh/id_rsa` or `~/.ssh/id_ed25519`.
    
        git config --global user.signingkey <path to private key>

3.  You can either turn on signing for all commits on your entire computer or just for some repositories.  
    For global enable, you can use
    
        git config --global commit.gpgsign true
    
    For local, change your directory to the Git repository and run:
    
        git config commit.gpgsign true

4.  Continue to “Sharing your key”.

### Exporting your GPG public key

1.  Export your public key: `gpg --armor --export <key> --output <public key filename>`

2.  Continue to “Sharing your key”.

### Sharing your key

1.  Take a hash of your **public** key.
    
        shasum -a 256 <path to PUBLIC key>

2.  Share your **public** key on the team channel in Signal.

3.  Verify your **public** key’s hash on video.

4.  Add all your contributors' SSH public keys to `~/.config/git/allowed_signers`. You will probably need to create that directory.
    
        mkdir -p ~/.config/git && touch ~/.config/git/allowed_signers

5.  Tell Git about that file:
    
        git config \
          --global \
          gpg.ssh.allowedSignersFile \
          "$HOME/.config/git/allowed_signers"

6.  For contributors who have GPG keys, download their keys and run the following command for each key:
    
        gpg --import <public key file>

## Everyone: Scripts

This uses several commands. To get these set up, add this script to your path as `git-pack-flow` and set it as executable.

    #!/usr/bin/env bash
    
    set -euo pipefail
    
    main()
    {
        if [[ $# -eq 0 ]]; then
            usage
            exit 1
        fi
    
        while true; do
            case "$1" in
                -d|--debug)
                    set -x
                    shift ;;
                -h|--help)
                    usage; exit 0 ;;
                *) break ;;
            esac
        done
    
        case $1 in
            "load")
                shift
                cmd_load "$@"
                ;;
            "save")
                shift
                cmd_save "$@"
                ;;
            *)
                echo "Invalid command provided."
                usage
                exit 1
        esac
    }
    
    cmd_load()
    {
        local incoming_branch
        incoming_branch="$(
            git bundle unbundle "$1" | \
                sed 's_.*refs/heads/__'
        )"
    
        if git branch | grep -q "$incoming_branch"; then
            git checkout "$incoming_branch"
        else
            git checkout -b "$incoming_branch"
        fi
    
        git fetch "$1" "$incoming_branch":
        git merge FETCH_HEAD
    
        echo "Loaded $incoming_branch"
    }
    
    cmd_save()
    {
        local branch
        branch="$(git rev-parse --abbrev-ref HEAD)"
    
        local escaped_branch
        escaped_branch="$(
            echo "$branch" | sed 's/[\/]/--/g'
        )"
    
        local now
        now="$(date -u "+%Y%m%dT%H%M%SZ")"
    
        local dest="$1/${escaped_branch}__${now}.pack"
    
        local commits
    
        if [[ "$branch" == "main" ]]; then
            commits="main"
        else
            commits="main..${branch}"
        fi
    
        git bundle create "$dest" "$commits"
    
        echo "Created bundle for branch \"$branch\":"
        echo "$dest"
    }
    
    usage()
    {
        echo -e "Usage: $(basename "$0") [options] [command]"
        echo -e ""
    
        echo -e "Options include:"
        echo -e "   -d --debug \t\t print commands in script as they"
        echo -e "              \t\t run"
        echo -e "   -h --help  \t\t print this help"
        echo -e ""
    
        echo -e "Commands:"
        echo -e "   save [dir] \t\t creates a bundle for the current"
        echo -e "              \t\t branch in dir"
        echo -e "   load [file]\t\t loads a branch bundle file"
        echo -e ""
    }
    
    main "$@"

## Lead: Setup

### Per Repository Setup

1.  Create a local clean clone of the repository on your local.
    
        mkdir signal-repos
        cd signal-repos/
        git clone <path to>

2.  Set that repository to require signatures on merge.
    
        git config merge.verifySignatures true

3.  Create a Signal channel for each repository.

4.  Compress the entire clean repo into a tarball.

5.  Post the tarball on the repository Signal channel.

## Developers: Development

1.  Download the latest main bundle from the repo Signal channel.

2.  Load the main bundle and checkout main. `$BUNDLE` is the downloaded bundle file.
    
        git-pack-flow load $BUNDLE

3.  Create a branch and perform development.

4.  Pick directory to put your outgoing bundles. In scripts, I call this `$BUNDLE_OUTBOX`.

5.  When ready for code review, create a bundle for your branch:
    
        git-pack-flow save $OUTBOX

6.  Send the file to the repo Signal channel with a title, description, and screenshots. Tag your reviewers.  
    The first lines of the message should be titled in the format of:
    
        PR
        fix(component): insert description here (ABCD-XXXX)

7.  Work with your reviewers via DM and repeat this process until your branch is approved.

## Developers: Review

1.  Download the latest bundle for a branch from the repo channel.

2.  From your repo directory, add the bundle commits into your repo, where `$INCOMING` is the downloaded bundle.
    
        git-pack-flow load $INCOMING

3.  Review the changes. If any changes are needed, work with the developer over DM and repeat loading as necessary.

4.  Sign off on the branch. Ensure that your signature is set up.
    
        git commit --signoff --allow-empty

5.  Create the bundle.
    
        git-pack-flow save $BUNDLE_OUTBOX

6.  Post the signed bundle to Signal with the leading lines:
    
        Approved
        fix(component): insert description here (ABCD-XXXX)

## Lead: Merge

1.  Download the latest bundle for a branch from the repo channel.

2.  Add the bundle to your local.
    
        git-pack-flow load $INCOMING

3.  Checkout `main`.

4.  Ensure that all the commits are ready and the sign-offs you expect are there.
    
        git log $BRANCH --show-signature

5.  Merge the branch into `main` with verification. Use our standard format for commit messages (e.g. `fix(component): description here (ABCD-1234)`)
    
        git checkout main
        git merge --verify-signatures $BRANCH

6.  Run lints, tests and the build to ensure the merge was clean.

7.  Create the build artifact:
    
    1.  If this is a package in a Rush repository, run this.
        
            rm -rf ../../common/temp/artifacts/packages/* && \
            rush rebuild && \
            rush publish \
                --pack \
                --include-all \
                --publish \
                --apply \
                --apply-git-tags-on-pack
        
        The resulting tarballs will be in `common/temp/artifacts/packages/`.
    
    2.  Many of our Next.js applications build with `pnpm run export` or `yarn run export` to `out/`.

8.  If changes were made while creating the build artifact, like version updates, commit those to `main`.

9.  Bundle up main:
    
        git-pack-flow $OUTBOX

10. Post the resulting bundle and artifacts to the repo channel. The first lines of the message should be in the format:
    
        MAIN
        fix(component): insert description here (ABCD-XXXX)
        by <PR author>
