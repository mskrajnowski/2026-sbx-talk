# docker sandboxes

Talk about [Docker Sandboxes](https://www.docker.com/products/docker-sandboxes/) (`sbx`) for sandboxing coding agents using micro VMs.

- https://www.docker.com/products/docker-sandboxes/
- https://docs.docker.com/ai/sandboxes/

## Why sandbox?

I don't feel safe enough running LLMs directly on my machine, it's like giving open access to your computer to a random person on the internet that promises they will not do anything wrong. Pinky swear.

The usual way coding agents try to protect the user from dangerous commands is that they will ask the user for approval of each command the LLM wants to make. I don't think this is a viable approach:

1. An average user won't even know what the command does, they have no way to audit it.
2. Even if you know what it does, it's extremely annoying to have to approve/deny every command.
3. You cannot leave the agent working in the background and do your own thing, babysitting the agent is just a giant waste of time.

Because of this a lot of people just run their coding agents in yolo mode, e.g. `claude --dangerously-skip-permissions`, directly on their machine.

## Why `sbx`?

The main points that lead me to trying out Docker Sandboxes is that it is focues around coding tasks and that it uses Virtual Machines for sandboxing.

There's a lot of tools that use docker/containers for sandboxing, but that cuts out the possibility of using containers inside of the sandbox, which is crucial in many of our projects that are based on docker compose for example. Sure, you could pass the docker socket to the sandbox, but at that point you're basically giving root access to the host.

On top of managing the VMs `sbx` also gives a reasonably easy way to setup the sandbox:

- set which IPs and domains the sandbox can access, e.g. remove access to the local network
- pass credentials for external services, e.g. pass LLM provider API token
- setup filesystem sharing, e.g. mount a volume with data that is not pushed to the repo

I also wanted to try out the `sbx` [clone mode](https://docs.docker.com/ai/sandboxes/workflows/#clone-mode), which does not simply mount the host project directory in a sandbox, instead it clones the project repository, so the agent has their own workspace and you can continue working without the agent messing in your filesystem. More on that [later](#clone-mode).

## Setting up `sbx`

https://docs.docker.com/ai/sandboxes/get-started/#install-and-sign-in

### `sbx login`

The most annoying thing about setting up `sbx` is that you have to sign in to a Docker account to do anything. The login process itself is not that annoying, you just run `sbx login`, it pops up a browser tab where you can sign in to Docker and you're done. However, at least on my Fedora KDE machine, `sbx` constantly forgets my credentials.

```sh
$ sbx version
sbx version: v0.34.0 2eae0c4fc3894475da3318615f69783b0e7be747
$ sbx ls
Your Docker session has expired. Starting the sign-in flow...

Your one-time device confirmation code is: XXXX-XXXX
Open this URL to sign in: https://login.docker.com/activate?user_code=XXXX-XXXX

By logging in, you agree to our Subscription Service Agreement. For more details, see https://www.docker.com/legal/docker-subscription-service-agreement/

Waiting for authentication...
```

There's no reason why listing my sandboxes should require phoning home and checking if I can do that.

Docker justifies the need to log in with their enterprise policy integration. An admin can setup company-wide network or filesystem policies, so that all developers automatically have it set up for them... For me this is an anti-feature, becase these company-wide policies overwrite your local ones, so if you are added to a Docker organization with such policies, you cannot configure `sbx` yourself.

Some "fun" side-effects of credential expiry is that the agent harness suddenly stops working properly, e.g. the TUI stops scaling well to the terminal window. If you run `sbx login` again and then resize the terminal, suddenly it works ok... for a while.

### Network policy

When you first set up `sbx` you will have to set up the default network policy:

- Allow all - allows any network requests, you need to explicitly deny access to any domains/IPs the sandbox should not have access to
- Deny all - denies all network requests, you need to explicitly allow access to any domains/IPs the sandbox should have access to
- Balanced - `sbx` has a predefined list of common domains that are needed to install packages or to call LLMs, which are allowed by default, anything else needs to be explicitly allowed like in "Deny all" mode

To enforce any policies you set up `sbx` runs a proxy that the sandbox routes all the network traffic through. You can use the `sbx` TUI to monitor what domains were requested, whether they were allowed or denied and to change the policy globally or per sandbox.

Personally I've landed on allow all with explicit deny rules to disallow access to the host and to the local network, so the agent can freely access the internet, but it can't snoop around my local network. I've started with deny all, but having to approve all the domains needed to install anything or fetch documentation was too much of a chore. I might use a much stricter policy when creating a sandbox that has access to more personal information, e.g. my email inbox or notes.

## Setting up a sandbox

`sbx` has built-in support for a couple agent harnesses, e.g. Anthropic Claude Code, OpenAI Codex, OpenCode. Full list can be found at https://docs.docker.com/ai/sandboxes/agents/

Starting a sandbox is as simple as:

```sh
sbx run opencode
```

which will:

- pull the opencode template (container image) from [Docker Hub](https://hub.docker.com/r/docker/sandbox-templates/tags)
- create and start a VM from the container image
- mount the current directory in the VM
- start `opencode` in the current directory from within the VM
- attach to the `opencode` TUI

`sbx` calls this the [Direct mode](https://docs.docker.com/ai/sandboxes/workflows/#direct-mode), since the agent has direct access to the project directory through a volume.

## Clone mode

```sh
sbx run --clone opencode
```

The other workflow is [Clone mode](https://docs.docker.com/ai/sandboxes/workflows/#clone-mode), where the repository is cloned into the VM, so that the agent has their own working copy.

To exchange changes with the sandbox, `sbx` sets up remotes on both sides:

- the host repo gets a `sandbox-{name}` remote that points to a [git daemon](https://git-scm.com/docs/git-daemon) running in the VM
- the sandbox repo gets an `origin` remote that is either pointing at `/run/sandbox/source` or if the host repo has an `origin` remote, it just copies the remote url to the sandbox

This way you can work on the host while the agent does something else in the background. Once it is done, you can `git fetch` and/or `git pull` its changes and hopefully eventually merge them. You could also set up the agent to open pull/merge requests in the actual repository if feeling brave.

### `/run/sandbox/source`

Some eagle eyed readers might have noticed something above, what is `/run/sandbox/source`? Well, in clone mode, `sbx` mounts your project directory as read-only to `/run/sandbox/source`, which is a bit of a problem.

The whole reason I wanted to use clone mode is to isolate the agent from my local filesystem. Mounting it read-only is a bit better than nothing, but it still might expose my local secrets, e.g. my local `.env`, this way. Sadly Docker has marked [an issue that reported this](https://github.com/docker/sbx-releases/issues/260) as working as intended...

Due to this I'm probably not going to use clone mode directly on my host repos. I will probably set up a clean clone for each project I want an agent to work on and start a sandbox with that, either using the direct or clone mode.

## Credentials

The agent probably won't be able to do much without some credentials, starting with the LLM provider credentials, so the agent can actually call the LLM it is based on.

`sbx` provides [facilities](https://docs.docker.com/ai/sandboxes/security/credentials/) to pass credentials without exposing them to the sandbox. The main idea is that all requests from the sandbox are routed through a proxy, where `sbx` can replace credential placeholders with real ones:

- The sandbox has a placeholder value, by default `proxy-managed`, which it includes, e.g. in an HTTP Authorization header
- The proxy has a list of credentials for different domains
- Whenever a request is sent that is matching one of the credential domains, the proxy looks for the placeholder and replaces it with the real value

There's two ways to create credentials `sbx secret set` and `sbx secret set-custom`, the latter marked as experimental.

With `sbx secret set` you only provide the name of the secret and the value, the actual replacement logic is set up in an `sbx` [kit](https://docs.docker.com/ai/sandboxes/customize/kits/#authenticate-to-external-services), e.g.

```sh
# set secret for all sandboxes
sbx secret set -g opencode
# sec secret for a named sandbox
sbx secret set NAME opencode
```

```yaml
# kit spec.yaml
network:
  # define a secret
  serviceDomains:
    opencode.ai: opencode
  # setup replacement logic
  serviceAuth:
    opencode:
      headerName: Authorization
      valueFormat: "Bearer %s"

# optional, inject the placeholder through an environment variable
environment:
  proxyManaged:
    - OPENCODE_API_KEY
```

IMHO, this is too much work to pass a secret. There's also no way to pass the placeholder value, it will always use `proxy-managed`, which might not work for tools that validate credentials before sending them, e.g. a library might check that an API key has to start with `sk-` in which case you can't use this flow.

This is why I personally use `sbx secret set-custom`, which let's you specify everything in one place and without creating/modifying a kit spec:

```sh
sbx secret set-custom -g \
  --host opencode.ai \
  --placeholder sk-opencode \
  --env OPENCODE_API_KEY
```

`--placeholder` and `--env` are optional. You can also generate random placeholders by including `{rand}` in the placeholder, e.g. `--placeholder 'sk-{rand}'`.

## Customizing the sandbox

To customize what's available in the sandbox you can:

- build a custom container image, which you can use as a [template](https://docs.docker.com/ai/sandboxes/customize/templates/)
- start a sandbox from a default template, modify it with `sbx exec` as needed and create a [template from a snapshot](https://docs.docker.com/ai/sandboxes/customize/templates/#saving-a-sandbox-as-a-template)
- prepare a [kit](https://docs.docker.com/ai/sandboxes/customize/kits/), which can modify a sandbox based on a default template

Personally I've opted for the kit approach as I didn't need to make too many changes to the environment. With a kit you can:

- set up environment variables
- set up files that should be automatically copied to either the sandbox project directory or the sandbox home directory, useful for adding tools configuration
- run commands at sandbox creation, useful to install tools
- run commands at sandbox startup, useful for running background services
- set netwoork policy for a sandbox
- set up the base template image and command to run the agent

You can check my [current kit](https://github.com/mskrajnowski/dotfiles/tree/40cbbc3719aacef57a0e7020b64a8e396b780847/sbx/.config/sbx-kit) for an example.

### Closed source

Sadly the source for `sbx`, the default templates and kits is not available publicly, so its not exactly clear what you are getting.

You can pull the images and inspect them, you can also review the commands that created each layer to reverse-engineer a `Dockerfile`, e.g. [docker/sandbox-templates:opencode-docker
](https://hub.docker.com/layers/docker/sandbox-templates/opencode-docker/images/sha256-699739ce96cc51e8638795d29359a0cc07aab837e595a8d67d6c0d645b29c0c5), but if you wanted to build a template and kit from scratch there's nothing to base on.

### Kit quirks

The order of operations when setting up a sandbox with a kit is a bit odd in my opinion.

`install` commands run before `initFiles` are added and static files are copied. This means you can't for example run `mise install` during the `install` phase if you want to provide `mise` configuration through static files.

Sandbox does not wait for `startup` commands to finish before starting the entrypoint (agent harness), regardless if they are marked as `background` or not. So if you want to, e.g. run `mise install`, before running the agent harness, so that the agent has access to all the tools needed, you are out of luck. This is particularly annoying to debug, because `mise install` will run, so the second time you run the agent it will work fine.

Because of these I ended up using `mise exec ...` as the entrypoint.

## My local setup

[mskrajnowski/dotfiles/sbx](https://github.com/mskrajnowski/dotfiles/tree/40cbbc3719aacef57a0e7020b64a8e396b780847/sbx)

## Summary

Pros:

- uses VMs for isolation
- docker is pre-installed in the VM
- credentials are not exposed
- network policy is easy to set up

Cons:

- closed source, especially for a security-related tool
- `/run/sandbox/source` exposes potentially sensitive information to the VM
- requires a Docker account
- broken Docker account credential management
- policy inherited from Docker organization
- using a custom local container image might be easier
- weird kit behaviors
