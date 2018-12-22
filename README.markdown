# Bootstrap CentOS VMs for training
Packer build scripts and tools for master, training, or learning VMs, and classroom_in_a_box setup.

See versioned release notes at [ReleaseNotes.md](ReleaseNotes.md).

## Usage

1. If you are looking for info on `classroom_in_a_box`, look in that directory for a [README.md](classroom_in_a_box/README.md) file for instructions on how to set up a classroom_in_a_box machine.
2. Install virtualbox, packer, vmware ovftool, bundler, and Xcode Command-Line Tools and ensure the binaries are in the path.
    * Install Bundler
      * `sudo gem install bundler`
    * Install XCode Command-Line Tools
      * `sudo xcode-select --install`
      * `sudo xcodebuild -license`
3. Run setup task to download and deploy base images:
    * `bundle install`
    * `rake setup`
4. Builds are triggered by a rake task that handles local caching and wraps the
packer command. To begin a build, run the corresponding rake task, for example
`rake build_base['master']`. (Details on the available build tasks are included below.)
5. To specify which version of PE to use for a base build, use the PE\_FAMILY,
PE\_VERSION, and PRE\_RELEASE environment variables, for example,
`PE_FAMILY=2016.3 PRE_RELEASE=true rake build_base['learning']`. See the *Environment variables*
section below for details on how the correct version is derived from these
variables.

## Detailed Usage

### Rake tasks

The Rakefile includes rake tasks to build the following VMs:

* training
  * `build_base['training']`
  * `build_main['training']`

* master
  * `build_base['master']`
  * `build_main['master']`

* learning
  * `build_base['learning']`
  * `build_main['learning']`

In addition, there are several tasks to help with caching, packaging, and
related processes:

* `cache_pe_installer`
* `package_learning`
* `ship['learning']`

### Environment Variables

The version of PE cached and subsequently installed can be specified through a
combination of three environment variables:

* **PE_VERSION**  
  This specifies the verison of a PE release or pre-release to be cached and/or
  used in a base build, for example `2016.2.0` for a release build or
  `2016.3.0-rc1-141-g5e261dc` for a pre-release build. If this is unset or
  set to `latest`, the version will default to the most recent release version
  in the case of a release build or the latest pre-release build of the
  specified family. (Note that currently setting `latest` for a release
  build will always use the latest release version regardless of a specified
  PE_FAMILY variable.)
* **PE_FAMILY**  
  Thos specifies a y-level release, such as `2016.2`. This can be specified
  instead of PE_VERSION to get the latest pre-release for the specified family.
* **PRE_RELEASE**  
  If this is set to true, rake will use pre-release PE versions.

* **STABLE_MODULES**
  If this is set to true, rake will use the "release" branch of the pltraining
  modules. Otherwise it defaults to using the "master".
 
A build without PE_VERSION or PE_FAMILY specified will use the latest release
version of PE. A build with PRE_RELEASE set to true requires PE_VERSION or
PE_FAMILY to be specified.

These environment variables are used to calculate corresponding values that are
passed through to the packer command in the format `-var name=value`. This
method of specifying variables takes precedence over variables specified
elsewhere, so the version information set via environment variables will take
precedence over any specified version in the build template.

* **AUTOMATED_BUILD**
  If this is set to `true`, the build will skip the bailout message after the
  build validation output.

### Packer

The build tasks in the Rakefile wrap the `packer` tool, which is used to
run the VM builds. Packer can also be run directly.

Packer scripts are provided in the `templates` directory. These depend on
packer (>= v0.10.0), VirtualBox, and ovftool.

The base VMs are the published puppetlabs vagrant boxes.  To download and
prepare them, run `setup.sh`. This will create the output, file_cache, and
packer_cache directories if they don't exist.  

The file_cache directory has a subfolder called "installers" for holding 
PE installer tarballs and another called gems for caching gems, the setup 
script will create any necessary directories if they don't exist. If you'd like
to keep those on a separate volume to save disk space, create symlinks before 
running the setup script. The installer is automatically cached by the
`cache_pe_installer` rake task, which precedes all base build tasks and selects
a version of PE based on provided environment variables.

The common configuration options for the training, learning, and master VMs
have been set up in educationbase.json and VM-specific variables are set in
*VMNAME*.json

After the base VM is provisioned according to the settings in *VMNAME*.json, the
bootstrap can be applied using educationbuild.json.

First create a base VM without any bootstrap applied:
- `packer build -var-file=templates/learning.json templates/educationbase.json`

To initiate a packer build of the learning vm on the base vm:
- `packer build -var-file=templates/learning.json templates/educationbuild.json`

For the training vm follow the same two steps but with training.json:
- `packer build -var-file=templates/training.json templates/educationbase.json`
- `packer build -var-file=templates/training.json templates/educationbuild.json`

There are also several AMI builds in templates. These don't require a base build.
To build these, set up the AWS builder requirements as described in the packer
documentation and use the following command:
- `packer build -var-file=templates/learning.json templates/awsbuild.json`

## Vagrant
There is a Vagrantfile that automates this process and builds on the
puppetlabs/centos-7.2-x86_64-nocm base box.

The Vagrant boxes will use the current local files for building rather than 
checking out the code from GitHub. They have full write access, to be aware that
new files in the repo be created and existing files may change.

There are three boxes specified.

To start a student vagrant box:
- `vagrant up`

To start a training vagrant box for instructor use:
- `vagrant up training`

To start a learning vagrant box:
- `vagrant up learning`


## Building with pre-released modules and code
To develop the VMs, you need to use modules and other code that haven't yet
been merged to master or released to the Puppet Forge.

The modules used during the VM build are included in `build_files/Puppetfile`.
See the documentation [here](https://puppet.com/docs/pe/latest/code_management/puppetfile.html#creating-and-editing-puppetfiles)
for detailed information on specifying git refs in a Puppetfile. 

## Troubleshooting

### Setup script
If the initial setup script fails, delete the contents of `output` and rerun.

### Failed builds
VM builds will fail if the output artifacts already exist.  You can avoid that
by using the `--force` flag with packer.  If a build does fail on the OVA export
you can run the ovftool directly.  For example usage see `scripts/export-ova.sh`.

