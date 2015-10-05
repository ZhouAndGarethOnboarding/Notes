# Onboading in Cloud Foundry
## Staying up to date

As cloudfoundry developers, we have to keep up to date with what's happening in
the CF world. At a minimum, we should be subscribed to the `cf-dev` and
`cf-bosh` mailing lists. You can see or sign up to these lists
[here](https://lists.cloudfoundry.org/). After subscribing to the lists, you
should receive a welcome email from the organization.

## Learning Git

Our first task was to learn how to use git as a part of a pairing workflow. We
added our email addresses to the `~/.git-authors` file:

~~~~
  gds: Gareth Smith; gsmith
  zy: Zhou Yu; zyu
email:
  domain: pivotal.io
~~~~

...and we got in the habit of typing `duet zy gds` (and using Gareth's ssh
keys) or `duet gds zy` (and using Zhou's ssh keys) at the beginning of each
day. We use ssh-keys on tiny USB sticks, as recommended
[here](http://tammersaleh.com/posts/building-an-encrypted-usb-drive-for-your-ssh-keys-in-os-x/)

## Local community

To keep up with other local cloud foundry users we joined
[three](http://www.meetup.com/London-Cloud-Foundry-User-Group-LCFUG-Meetup/)
[meetup](http://www.meetup.com/London-PaaS-User-Group-LOPUG/)
[groups](http://www.meetup.com/London-Pivotal-Cloud-Native-Apps-Meetup)

## We can install BOSH

Since we're working on a machine that already has BOSH installed, we decided to work in a VM. 

We ran:

~~~~
$ vagrant init hashicorp/precise32
$ vagrant up
$ vagrant ssh
~~~~

This gave us a shell on a brand new ubuntu box, in which we could go through
all the details of installing BOSH firsthand.

inside the resulting VM, we ran followed the instructions
[here](http://bosh.io/docs/bosh-cli.html) and ran the following:

~~~~
sudo gem install bosh_cli bosh_cli_plugin_micro --no-ri --no-rdoc
~~~~

but we ran into the problem of not having the `sqlite3` package. So we ran:

~~~~
sudo apt-get update
sudo apt-get install sqlite3 libsqlite3-dev build-essential ruby1.9.1 ruby1.9.1-dev python-software-properties
sudo add-apt-repository ppa:brightbox/ruby-ng-experimental
sudo apt-get update
sudo apt-get install -y ruby2.0 ruby2.0-dev ruby2.0-doc
~~~~

and then re-ran:

~~~~
sudo gem install bosh_cli bosh_cli_plugin_micro --no-ri --no-rdoc
~~~~

Now we're able to run `bosh help` -- we have the bosh CLI installed!

## Deploying BOSH Lite

...in the same vagrant box, we followed the instructions
[here](https://github.com/cloudfoundry/bosh-lite/blob/master/README.md),
beginning with `sudo apt-get install vagrant`

Unfortunately, the version in the repo was pretty old. To head off any problems early, we downloaded the latest deb package from vagrantup.com:

~~~~
sudo dpkg -i /vagrant/vagrant_1.7.4_i686.deb
sudo apt-get install git
git clone https://github.com/cloudfoundry/bosh-lite
cd bosh-lite
sudo apt-get install linux-headers-generic virtualbox-ose-dkms
sudo apt-get source linux-image-$(uname -r)
sudo apt-get install linux-headers-`uname -r`
sudo apt-get remove virtualbox-ose-dkms
sudo apt-get install virtualbox-ose-dkms
~~~~

Even after all this, our attempts to configure virtualbox to prepare it for running vagrant and bosh result in mysterious error messages:

~~~~
$ sudo dpkg-reconfigure virtualbox-dkms

------------------------------
Deleting module version: 4.1.12
completely from the DKMS tree.
------------------------------
Done.
Loading new virtualbox-4.1.12 DKMS files...
Building only for 3.2.0-23-generic-pae
Module build for the currently running kernel was skipped since the
kernel source for this kernel does not seem to be installed.
 * Stopping VirtualBox kernel modules
   ...done.
 * Starting VirtualBox kernel modules
 * No suitable module for running kernel found
   ...fail!
invoke-rc.d: initscript virtualbox, action "restart" failed.
$
~~~~

At this point, we realised that [this might not be
possible](https://github.com/mitchellh/vagrant/issues/572), and our anchor said
"at least in the past, bosh-lite wasn’t even super-stable on linux, let alone
inside vagrant.."

So we switched to working locally on the mac.

### Take 2

Outside the vagrant box now...

~~~~
cd ~/workspace/
git clone https://github.com/cloudfoundry/bosh-lite bosh-lite-exercise
cd bosh-lite-exercise
vagrant up
export no_proxy=xip.io,192.168.50.4
bosh target 192.168.50.4 lite
~~~~

And we login into BOSH by (admin/admin as the credentials):
~~~~
bosh login
~~~~

and (this might ask you for the password for your machine)
~~~~
bin/add-route
~~~~

...to set up network routes by which the local bosh-cli can access the local
bosh-lite inside the virtualbox.

~~~~
vagrant box update
vagrant destroy
vagrant up
~~~~

...to upgrade the VM to the latest version to run with bosh lite.

Now (so long as you didn't accidentally deploy bosh lite on a machine that was already running bosh lite -- hahaha -- who would do such a thing?!?!) you can do:

~~~~
$ bosh vms
No deployments
~~~~

## Deploying Cloudfoundry to our BOSH-LITE

Following instructions from [here](https://docs.cloudfoundry.org/deploying/boshlite/deploy_cf_boshlite.html)

~~~~
cd DeployingCFToBoshLite
bosh upload stemcell ~/Downloads/bosh-stemcell-2776-warden-boshlite-ubuntu-trusty-go_agent.tgz
echo "---">> manifest-stub.yml
echo "director_uuid: `bosh status --uuid`" >> manifest-stub.yml
cd ~/workspace
git clone https://github.com/cloudfoundry/cf-release.git
cd cf-release
./scripts/update
~~~~

At this point, the official docs we're following instruct us to install
[spiff](https://github.com/cloudfoundry-incubator/spiff). But the readme of
spiff tells us that development is paused, including pull requests. Is there a
new better way of doing things? Or is this all fine? We don't know, so we're
carrying on with Spiff.

After successfully intalling the spiff, we did 

~~~~
bosh target 192.168.50.4
./scripts/generate-bosh-lite-dev-manifest  ~/Documents/OnBoardingBlog/DeployingCFToBoshLite/manifest-stub.yml
bosh create release
bosh upload release
bosh deploy
~~~~

Which didn't go as well as we'd hoped:

~~~~
Getting deployment properties from director...
Unable to get properties list from director, trying without it...
Compiling deployment manifest...
Release 'cf' not found on director. Unable to resolve 'latest' alias in manifest.
~~~~

We think this is because we chose a non-standard name when we did
bosh-create-release (cf-onboarding-gareth-zhou instead of cf). So we made a
note to figure that out later -- for now we're re-trying with the default name.

Re-running `bosh create release` suprised us by continuing to use the name we
picked last time we ran it, and not offering a chance to change it. We observed
that configs/dev.yml contained our non-standard name, and was git ignored,
while configs/final.yml contained the default name and was checked in to git.
So we removed configs/dev.yml and reran `bosh create release` and thankfully
the CLI prompted us to input a name for this project. This time, `bosh deploy`
ran successfully, and helpfully informed us that we were creating a new deploy
for the first time... At this point the machine hung. So we hard-rebooted and
started again, but with fewer VMs running... And crucially, without allowing
any one of them to eat all our RAM:

~~~~
export VM_MEMORY=8192
bosh deploy
~~~~

IT WORKS!

We are now able to test our deployment:

~~~~
cf api --skip-ssl-validation https://api.10.244.0.34.xip.io
Setting api endpoint to https://api.10.244.0.34.xip.io...
FAILED
Error performing request: Get https://api.10.244.0.34.xip.io/v2/info: dial tcp 10.244.0.34:443: i/o timeout
~~~~

....did the docs tell us to use the wrong IP?

No. Actually, we just needed to wait 10 minutes, and then everything worked.

~~~~
cd $BOSH-LITE
./bin/add-route # this should have been done before, if you didn't crash your machine with an enormous VM like we did.
cd -
cf api --skip-ssl-validation https://api.10.244.0.34.xip.io
cf auth admin admin
cf create-org test-org
cf target -o test-org
cf create-space test-space
cf target -s test-space
~~~~

## As a Pivot, I can customise the CF release

Find out where your config file is with `bosh deployment` ; edit that file ;
`bosh deploy`. Boom.

## Deploying a CF app to our local CF release

Our first problem seems to be that we don't have working java dev tools on this machine... But we're not sure. It's confusing.

We know that when we tried to push the demo app to our local CF, we got
buildpack errors. `cf push` said that no buildpack recognised our app. Trying
to force a java pack caused different unhelpful errors.

Trying to push a (simpler?) java example app taught us that we don't have a
working `native2ascii` install.

Finally, we decided to try pushing the demo app to PWS. We created a free account, and it worked fine.

```
cf push
```

As the README predicted, this worked, but without a rabbitmq service, it
doesn't do much. We weren't able to create a rabbitmq service, since rabbitmq
isn't available in the PWS marketplace (`cf marketplace`). We were able to create and destroy a
free `pubnub` service with `cf create=service pubnub free mypubnub` and `cf
delete-service mypubnub`. We could see which services we had available in our
org using `cf services`

Before switching back to the local CF, we got a list of which buildpacks PWS has: 

`cf buildpacks`

```
buildpack              position   enabled   locked   filename
ruby_buildpack         1          true      false    ruby_buildpack-cached-v1.6.7.zip
nodejs_buildpack       2          true      false    nodejs_buildpack-cached-v1.5.0.zip
java_buildpack         3          true      false    java-buildpack-v3.2.zip
go_buildpack           4          true      false    go_buildpack-cached-v1.6.0.zip
liberty_buildpack      5          true      false    liberty_buildpack.zip
python_buildpack       6          true      false    python_buildpack-cached-v1.5.0.zip
php_buildpack          7          true      false    php_buildpack-cached-v4.1.2.zip
staticfile_buildpack   8          true      false    staticfile_buildpack-cached-v1.2.1.zip
binary_buildpack       9          true      false    binary_buildpack-cached-v1.0.1.zip
```

....it turned out our CF was Just Bad. We destroyed and re-created it, and the
new one worked fine.

## Scaling a CF app

`cf scale APPNAME -i 2`

Boom.

## Deploying a thing to redis-labs CUPS

WTF is CUPS?

Redis-labs is a service: https://redislabs.com/

We created an account with Gareth's email address, and used that to create a "subscription" called `onboardingRedis` with pw `wtfisgoingon`

`cf cups myredis -p "host, port, dbname, password"`

## make a broker for this stuff

We figured out how to push a golang app to CF, but this turned out to be unnecessary.

We started test-driving a broker: http://github.com/ZhouAndGarethOnboarding/ReddisBroker


## As a Pivot, I can connect a sample ruby application to a redis-labs CUPS

We were given a link to a [github project](https://github.com/pivotal-cf/cf-redis-example-app)

This project has an excellent README, which guides you through deployment. For example, the first thing we did was follow these instructions:

~~~~
$ git clone git@github.com:pivotal-cf/cf-redis-example-app.git
$ cd cf-redis-example-app
$ cf push cf-redis-example-app --no-start
$ cf create-service p-redis development redis
$ cf bind-service redis-example-app redis
~~~~

Unfortunately, when we got to step three we hit our first snag:

~~~~
$ cf push redis-example-app --no-start
No API endpoint set. Use 'cf login' or 'cf api' to target an endpoint.

$ 
~~~~

It looks like we're not currently logged in to an instance of Cloud Foundry.
After a little flailing, we decided the simplest thing to do was to sign up for
a free trial of the Pivotal instance: [PWS](http://run.pivotal.io). Now we can
do the following:

~~~~
cf login -a https://api.run.pivotal.io
~~~~

and put in our email and password when the CLI asks.

Now we can carry on with our `cf push`:

~~~~
$ cf push cf-redis-example-app --no-start

Using manifest file /Users/pivotal/workspace/cf-redis-example-app/manifest.yml

Creating app cf-redis-example-app in org cf-onboarding / space development as zyu@pivotal.io...
OK

Creating route cf-redis-example-app.cfapps.io...
OK

Binding cf-redis-example-app.cfapps.io to cf-redis-example-app...
OK

Uploading cf-redis-example-app...
Uploading app files from: /Users/pivotal/workspace/cf-redis-example-app
Uploading 348.9K, 46 files
Done uploading
OK
~~~~

So, we've uploaded our tiny app to a cloudfoundry instance, but we haven't
started that app yet. If we try to access it through the web (using `curl`)
we'll get an error:

~~~~
$ export APP=cf-redis-example-app.cfapps.io

$ curl -X PUT $APP/foo -d 'data=bar'
curl: (6) Could not resolve host: redis-example-app.my-cloud-foundry.com
~~~~

...so we'd better bind and start the app:

~~~~
$ cf create-service p-redis development redis
Creating service redis in org cf-onboarding / space development as zyu@pivotal.io...
FAILED
Service offering p-redis not found
~~~~

...oops, it looks like the example app README had a typo in it. Let's try this instead:

~~~~
$ cf create-service cf-redis-example-app development redis
FAILED
Service offering cf-redis-example-app not found
~~~~

Hm.

Our PM (Will)[http://github.com/goonzoid] came to the rescue and explained that
p-redis is something that's set up on many Pivotal machines, but not on this
one. We'll come back to this later.
