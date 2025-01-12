---
title: "Advanced Usage of Apptainer"
teaching: 20
exercises: 20
questions:
- "How do I modify the default behavior of Apptainer when entering a container?"
- "How can I submit batch jobs when inside an Apptainer container?"
- "How can I use Apptainer when I'm already inside an Apptainer container?"
objectives:
- "Learn how to add custom prompts to your shell logon file and detect if you're inside a container."
- "Learn how to run commands on the host machine while still inside a container."
- "Learn how to tell if your Apptainer installation can run nested containers (Apptainer-in-Apptainer)."
---

# Login file customization for Apptainer

Most environment variables from the host machine and session will be automatically inherited when starting an Apptainer container.
However, there are some [exceptions](https://apptainer.org/docs/user/latest/environment_and_metadata.html#environment-from-the-host),
among them the `PS1` variable that specifies the shell prompt and optionally the terminal window title (see [here](https://wiki.archlinux.org/title/Bash/Prompt_customization) for a nice guide).
You can specify a custom command line prompt so it's easy to know at a glance if you're inside a container or not.

For example, these two lines give the prompt `[user@machine dir]$ ` on the host machine and `[user@machine dir *]$ ` in a container,
using the asterisk to indicate that you're in a container:

~~~bash
PS1="[\u@\h \W]\$ "
APPTAINERENV_PS1="[\u@\h \W *]\$ "
~~~

These settings can be placed in your `.bashrc` or `.bash_login` file.
You can also include an environment variable like `$APPTAINER_NAME` in the prompt or terminal window title,
which will display the name of the active container (if any).

# Script customization for Apptainer

Whether in your login file or another script, you might want to have certain operations that only execute if you're inside a container (or if you aren't).
You can achieve this with a simple check (replace `...` with your code):

~~~bash
if [ -n "$APPTAINER_CONTAINER" ]; then
    ...
fi
~~~

To reverse the check, use `-z` instead of `-n` ([guide to bash comparison operators](https://tldp.org/LDP/abs/html/comparison-ops.html)).

# Using batch systems with Apptainer

Most containers do not include the HTCondor executables or configuration files.
There are several options to work around this, in order to continue to use legacy job submission code that may depend on software that only runs on older operating systems (e.g. `CMSSW_10_6_X`).

1. Keep another terminal open in the host OS (outside of the container) and only execute batch commands there. This can be very tedious, so in general, it is *not recommended*.
1. Build a new container that includes HTCondor. An example of this option can be found at [https://gitlab.cern.ch/cms-cat/cmssw-lxplus](cmssw-lxplus) (aimed at CERN lxplus). While this allows the use of batch systems transparently, it requires exactly replicating the host machine's HTCondor configuration. It also does not scale well, because each possible container must be rebuilt whenever the container is updated, and potentially by each user (if the rebuilt containers are not distributed centrally). Therefore, in general, this option is *not recommended*.
1. Use the [`call_host` software](https://github.com/FNALLPC/lpc-scripts?tab=readme-ov-file#call_hostsh), which allows executing commands on the host OS while *still inside* the container. `call_host` is set up by default to handle HTCondor commands, as well as EOS filesystem commands, but it can handle any command (that doesn't require tty input).
1. Use the HTCondor Python bindings, which can be installed more easily inside most containers. The HTCondor configuration from the host OS must be correctly mounted inside the container. An example of this can be found at [lpcjobqueue](https://github.com/CoffeaTeam/lpcjobqueue). This option works very well for some operations, but if a site is using customized HTCondor scripts, the bindings will not pick up the customizations.

> ## Using containers in a batch job
> 
> HTCondor supports Apptainer ([documentation](https://htcondor.readthedocs.io/en/lts/admin-manual/singularity-support.html)). You can specify an Apptainer (or Docker) container that a job should use with the ClassAd:
> ~~~
> +SingularityImage = "..."
> ~~~
> Some sites also have site-specific ClassAds for you to specify just the target OS so it can find the right container automatically, but these are not standardized.
{: .testimonial}

> ## Going deeper
> 
> Read the instructions for the [`call_host` software](https://github.com/FNALLPC/lpc-scripts?tab=readme-ov-file#call_hostsh), install it on the cluster you usually use, and try it out. Besides HTCondor, what other commands are useful to call on the host?
{: .callout}

# Apptainer-in-Apptainer

There are several important cases where it may be necessary to invoke Apptainer when already inside an Apptainer container (nested Apptainer or Apptainer-in-Apptainer):
* Creating or using a container to preserve a workflow that involves the use of other containers
* Running batch jobs on the LHC computing grid: most batch jobs now start inside an Apptainer container to guarantee the user's choice of operating system, so any further container use is nested

This is possible, but there are certain system requirements that must be met:
1. The Apptainer configuration, by default at `/etc/apptainer/apptainer.conf`, must include the line `allow setuid = no` (and must not have `allow setuid = yes`).
1. The operating system must have unprivileged user namespaces enabled.

The CMS software provides a script to test for this behavior: [apptainer-check.sh](https://github.com/cms-sw/cmssw/blob/master/Utilities/ReleaseScripts/scripts/apptainer-check.sh).

If this behavior is not found on the machine you're using, there are a few options:
1. Ask your system administrator to enable the behavior, following the [Apptainer User Namespace guide](https://apptainer.org/docs/admin/latest/user_namespace.html).
1. Instead of using the system version of Apptainer, use the one distributed by the Open Science Grid on cvmfs at `/cvmfs/oasis.opensciencegrid.org/mis/apptainer/bin/apptainer`. This executable has its own configuration file that follows the above requirements (though it may still not work for nested Apptainer use in all cases, depending on the local system settings). To force the use of this version, you can modify your `PATH`: `export PATH=/cvmfs/oasis.opensciencegrid.org/mis/apptainer/bin:$PATH`.

{% include links.md %}
