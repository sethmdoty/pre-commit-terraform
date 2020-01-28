---
description: Cook your Own Packages
---

# FPM

Written by: Mathias Lafeldt Edited by: Joseph Kern 

### Introduction 

When it comes to building packages, there is one particular tool that has grown in popularity over the last years: fpm . fpm’s honorable goal is to make it as simple as possible to create native packages for multiple platforms, all without having to learn the intricacies of each distribution’s packaging format \(.deb, .rpm, etc.\) and tooling. With a single command, fpm can build packages from a variety of sources including Ruby gems, Python modules, tarballs, and plain directories. Here’s a quick example showing you how to use the tool to create a Debian package of the AWS SDK for Ruby : $ fpm -s gem -t deb aws-sdk Created package :path=&gt;”rubygem-aws-sdk1.59.0all.deb” It is this simplicity that makes fpm so popular. Developers are able to easily distribute their software via platform-native packages. Businesses can manage their infrastructure on their own terms, independent of upstream vendors and their policies. All of this has been possible before, but never with this little effort. In practice, however, things are often more complicated than the one-liner shown above. While it is absolutely possible to provision production systems with packages created by fpm, it will take some work to get there. The tool can only help you so far. In this post we’ll take a look at several best practices covering: dependency resolution, reproducible builds, and infrastructure as code. All examples will be specific to Debian and Ruby, but the same lessons apply to other platforms/languages as well. Resolving dependencies Let’s get back to the AWS SDK package from the introduction. With a single command, fpm converts the aws-sdk Ruby gem to a Debian package named rubygem-aws-sdk. This is what happens when we actually try to install the package on a Debian system:

```text
$ sudo dpkg —install rubygem-aws-sdk_1.59.0_all.deb
…
dpkg: dependency problems prevent configuration of rubygem-aws-sdk:
 rubygem-aws-sdk depends on rubygem-aws-sdk-v1 (= 1.59.0); however:
  Package rubygem-aws-sdk-v1 is not installed.
…
```

As we can see, our package can’t be installed due to a missing dependency \(rubygem-aws-sdk-v1\). Let’s take a closer look at the generated .deb file:

```text
$ dpkg —info rubygem-aws-sdk_1.59.0_all.deb
 …
 Package: rubygem-aws-sdk
 Version: 1.59.0
 License: Apache 2.0
 Vendor: Amazon Web Services
 Architecture: all
 Maintainer: 
 Installed-Size: 5
 Depends: rubygem-aws-sdk-v1 (= 1.59.0)
 Provides: rubygem-aws-sdk
 Section: Languages/Development/Ruby
 Priority: extra
 Homepage: http://aws.amazon.com/sdkforruby
 Description: Version 1 of the AWS SDK for Ruby. Available as both `aws-sdk` and `aws-sdk-v1`.
```

Use aws-sdk-v1 if you want to load v1 and v2 of the Ruby SDK in the same application. fpm did a great job at populating metadata fields such as package name, version, license, and description. It also made sure that the Depends field contains all required dependencies that have to be installed for our package to work properly. Here, there’s only one direct dependency – the one we’re missing. While fpm goes to great lengths to provide proper dependency information – and this is not limited to Ruby gems – it does /not/ automatically build those dependencies. That’s our job. We need to find a set of compatible dependencies and then tell fpm to build them for us. Let’s build the missing rubygem-aws-sdk-v1 package with the exact version required and then observe the next dependency in the chain:

```text
$ fpm -s gem -t deb -v 1.59.0 aws-sdk-v1
Created package {:path=>”rubygem-aws-sdk-v1_1.59.0_all.deb”}

$ dpkg —info rubygem-aws-sdk-v1_1.59.0_all.deb | grep Depends
 Depends: rubygem-nokogiri (>= 1.4.4), rubygem-json (>= 1.4), rubygem-json (<< 2.0)
```

Two more packages to take care of: rubygem-nokogiri and rubygem-json. By now, it should be clear that resolving package dependencies like this is no fun. There must be a better way. In the Ruby world, Bundler is the tool of choice for managing and resolving gem dependencies. So let’s ask Bundler for the dependencies we need. For this, we create a Gemfile with the following content: Gemfile

```text
source “https://rubygems.org”
gem “aws-sdk”, “= 1.59.0”
gem “nokogiri”, “~> 1.5.0” # use older version of Nokogiri
```

We then instruct Bundler to resolve all dependencies and store the resulting .gem files into a local folder:

```text
$ bundle package
…
Updating files in vendor/cache
json-1.8.1.gem
nokogiri-1.5.11.gem
aws-sdk-v1-1.59.0.gem
aws-sdk-1.59.0.gem
```

We specifically asked Bundler to create .gem files because fpm can convert them into Debian packages in a matter of seconds:

```text
$ find vendor/cache -name ‘*.gem’ | xargs -n1 fpm -s gem -t deb
Created package {:path=>”rubygem-aws-sdk-v1_1.59.0_all.deb”}
Created package {:path=>”rubygem-aws-sdk_1.59.0_all.deb”}
Created package {:path=>”rubygem-json_1.8.1_amd64.deb”}
Created package {:path=>”rubygem-nokogiri_1.5.11_amd64.deb”}
```

As a final test, let’s install those packages…

```text
$ sudo dpkg -i *.deb
…
Setting up rubygem-json (1.8.1) …
Setting up rubygem-nokogiri (1.5.11) …
Setting up rubygem-aws-sdk-v1 (1.59.0) …
Setting up rubygem-aws-sdk (1.59.0) …
```

…and verify that the AWS SDK actually can be used by Ruby:

```text
$ ruby -e “require ‘aws-sdk’; puts AWS::VERSION”
1.59.0
```

The purpose of this little exercise was to demonstrate one effective approach to resolving package dependencies for fpm. By using Bundler – the best tool for the job – we get fine control over all dependencies, including transitive ones \(like Nokogiri, see Gemfile\). Other languages provide similar dependency tools. We should make use of language specific tools whenever we can. Build infrastructure After learning how to build all packages that make up a piece of software, let’s consider how to integrate fpm into our build infrastructure. These days, with the rise of the DevOps movement, many teams have started to manage their own infrastructure. Even though each team is likely to have unique requirements, it still makes sense to share a company-wide build infrastructure, as opposed to reinventing the wheel each time someone wants to automate packaging. Packaging is often only a small step in a longer series of build steps. In many cases, we first have to build the software itself. While fpm supports multiple source formats, it doesn’t know how to build the source code or determine dependencies required by the package. Again, that’s our job. Creating a consistent build and release process for different projects across multiple teams is hard. Fortunately, there’s another tool that does most of the work for us: fpm-cookery . fpm-cookery sits on top of fpm and provides the missing pieces to create a reusable build infrastructure. Inspired by projects like Homebrew , fpm-cookery builds packages based on simple recipes written in Ruby. Let’s turn our attention back to the AWS SDK. Remember how we initially converted the gem to a Debian package? As a warm up, let’s do the same with fpm-cookery. First, we have to create a recipe.rb file:

```text
recipe.rb
class AwsSdkGem < FPM::Cookery::RubyGemRecipe
  name    “aws-sdk”
  version “1.59.0”
end
Next, we pass the recipe to fpm-cook, the command-line tool that comes with fpm-cookery, and let it build the package for us:
$ fpm-cook package recipe.rb
===> Starting package creation for aws-sdk-1.59.0 (debian, deb)
===>
===> Verifying build_depends and depends with Puppet
===> All build_depends and depends packages installed
===> [FPM] Trying to download {“gem”:”aws-sdk”,”version”:”1.59.0”}
…
===> Created package: /home/vagrant/pkg/rubygem-aws-sdk_1.59.0_all.deb
```

To complete the exercise, we also need to write a recipe for each remaining gem dependency. This is what the final recipes look like:

recipe.rb

```text
class AwsSdkGem < FPM::Cookery::RubyGemRecipe
  name       “aws-sdk”
  version    “1.59.0”
  maintainer “Mathias Lafeldt “

  chain_package true
  chain_recipes [“aws-sdk-v1”, “json”, “nokogiri”]
end
```

aws-sdk-v1.rb

```text
class AwsSdkV1Gem < FPM::Cookery::RubyGemRecipe
  name       “aws-sdk-v1”
  version    “1.59.0”
  maintainer “Mathias Lafeldt “
end
```

json.rb

```text
class JsonGem < FPM::Cookery::RubyGemRecipe
  name       “json”
  version    “1.8.1”
  maintainer “Mathias Lafeldt “
end
```

nokogiri.rb

```text
class NokogiriGem < FPM::Cookery::RubyGemRecipe
  name       “nokogiri”
  version    “1.5.11”
  maintainer “Mathias Lafeldt “

  build_depends [“libxml2-dev”, “libxslt1-dev”]
  depends       [“libxml2”, “libxslt1.1”]
end
```

Running fpm-cook again will produce Debian packages that can be added to an APT repository and are ready for use in production. Three things worth highlighting:

* fpm-cookery is able to build multiple dependent packages in a row \(configured by chain\_ attributes\), allowing us to build everything with a single invocation of fpm-cook.
* We can use the attributes build\_depends and depends to specify a package’s build and runtime dependencies. When running fpm-cook as root, the tool will automatically install missing dependencies for us.
* I deliberately set the maintainer attribute in all recipes. It’s important to take responsibility of the work that we do. We should make it as easy as possible for others to identify the person or team responsible for a package.

  fpm-cookery provides many more attributes to configure all aspects of the build process. Among other things, it can download source code from GitHub before running custom build instructions \(e.g. make install\). The fpm-recipes repository is an excellent place to study some working examples. This final example, a recipe for chruby , is a foretaste of what fpm-cookery can actually do:

recipe.rb

```text
class Chruby < FPM::Cookery::Recipe
  description “Changes the current Ruby”

  name     “chruby”
  version  “0.3.8”
  homepage “https://github.com/postmodern/chruby”
  source   “https://github.com/postmodern/chruby/archive/v#{version}.tar.gz”
  sha256   “d980872cf2cd047bc9dba78c4b72684c046e246c0fca5ea6509cae7b1ada63be”

  maintainer “Jan Brauer “

  section “development”

  config_files “/etc/profile.d/chruby.sh”

  def build
nothing to do here
  end

  def install
make :install, “PREFIX” => prefix
etc(“profile.d”).install workdir(“chruby.sh”)
  end
end
```

chruby.sh source /usr/share/chruby/chruby.sh Wrapping up fpm has changed the way we build packages. We can get even more out of fpm by using it in combination with other tools. Dedicated programs like Bundler can help us with resolving package dependencies, which is something fpm won’t do for us. fpm-cookery adds another missing piece: it allows us to describe our packages using simple recipes, which can be kept under version control, giving us the benefits of infrastructure as code: repeatability, automation, rollbacks, code reviews, etc. Last but not least, it’s a good idea to pair fpm-cookery with Docker or Vagrant for fast, isolated package builds. This, however, is outside the scope of this article and left as an exercise for the reader.

