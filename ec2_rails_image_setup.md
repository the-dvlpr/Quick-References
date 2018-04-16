# Punchlist for Serving your Rails App on an EC2 Instance
Run through this list to setup your EC2 instance and host your rails app. This list makes a few assumptions (make changes as necessary)
  * You have github repo with a rails app pushed to it that we'll just pull down to the new server
  * You're using Rails 5.1

If not, see very bottom for steps to build a quick example app.
---
### Launching EC2 instance
- [ ] Use the basic Linux AMI distro: *Amazon Linux AMI 2017.09.0 (HVM), SSD Volume Type - ami-8c1be5f6*
- [ ] Select a size: *t2.micro* is fine for this, unless you need larger for a production app
- [ ] Configure the security group like below so we can:
  * Have personal access to SSH into our EC2 instance
  * Have personal access to our instance's port 3000 for testing
  * And allow anyone to access port 80, where the rails app will be served   
  ![](https://s3.amazonaws.com/peronalgithub/Security+Settings+for+EC2+instance.png)
- [ ] Use existing pem file or create a new one (if creating a new one don't forget to navigate to directory and run `chmod 400 file_name.pem` to allow read permissions)
---
### Setting Up EC2 Instance Development Environment
- [ ] SSH into your instance
- [ ] $`sudo yum install git gcc openssl-devel readline-devel zlib-devel` to install libraries
- [ ] $`git clone https://github.com/rbenv/rbenv.git ~/.rbenv` to install rbenv
- [ ] $`echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bash_profile` to add rbenv path to bash
- [ ] $`~/.rbenv/bin/rbenv init`
- [ ] $`echo 'eval "$(rbenv init -)"' >> ~/.bash_profile` to initialize rebenv on bash start up
- [ ] $`eval "$(rbenv init -)"` to initialize rbenv
- [ ] $`source ~/.bash_profile`
- [ ] $`type rbenv` sanity check to test if rbenv is initialized (will return "rbenv is a function" if so)
- [ ] $`git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build`
- [ ] $`rbenv install 2.4.2` to install ruby-2.4.2 (this takes a moment)
- [ ] $`rbenv global 2.4.2` to set the ruby version for use
- [ ] $`ruby -v` sanity check to test version of ruby is 2.4.2
- [ ] $`gem install bundler`
- [ ] $`git clone git://github.com/dcarley/rbenv-sudo.git ~/.rbenv/plugins/rbenv-sudo`
- [ ] $`sudo yum groupinstall "Development tools" - to install gcc compiler`
- [ ] $`sudo yum install sqlite-devel` only if you're using sqlite. Skip if using mongo and remove sqlite from the gem file
- [ ] Make sure your gem file includes `gem 'execjs'` and `gem 'therubyracer', :platforms => :ruby` and your rails gem is set to 5.1 with `gem 'rails', '~> 5.1'`
- [ ] $`git clone #YOUR_REPOSITORY_URL`
- [ ] $`cd #YOUR_REPOSITORY`
- [ ] $`bundle`
- [ ] $`exec bash` to restart bash
- [ ] $`rails s -p 3000 -b 0.0.0.0` to launch server

Now navigate to the EC2 instance's public IP followed by port 3000 in your browser and you're all set!

**If you've got a domain purchased and want to host this site publicly in production**
If you've got a domain change the DNS @ record to point to the new server's IP address

Next add the following functions to your .bash\_profile (you'll have to do these every time you reboot the server so it's nice to save them as functions, unless you otherwise have them run each time the server starts up)
```
:redirect(){
    sudo iptables -t nat -I PREROUTING -p tcp --dport 80 -j REDIRECT --to-ports 3000
}

:launch(){
    nohup rails s -e production -b 0.0.0.0 -p 3000 &
}
```

Let me break these functions down:

:redirect reroutes all traffic coming in on port 80 to port 3000 where you'll have your server running.

:launch starts up your rails server. The `nohup` means no hangup and won't stop if you end your SSH connection with the server. `-e production` means you're running the production version of your app (leave this out if you haven't set up your production app sepearately but look into the differences). And the `&` at the end means run as a background operation so you can hit return and keep using the bash terminal. Now with nohup, output is automatically output to nohup.out so just vim or nano into that file to see the output of from the server. `:e` in vim refreshes while in the file and `shift + g` brings you automatically to the end of the file -- you'll find these useful.

So now just run the functions
- [ ] $`:redirect`
- [ ] $`:launch`

and you should be able to navigate to the website's domain and see your newly hosted rails app live.

---
### Rails App
Create your rails app on your home computer and push to github
- [ ] $`rails _5.1_ new app_name`
- [ ] Navigate to the app's gem file and make sure you have the gems as specified above
- [ ] $`bundle update`
- [ ] $`git init`
- [ ] $`git add .`
- [ ] $`git commit -m 'Initial commit'`
- [ ] $`git remote add origin #YOUR_REPOSITORY`
- [ ] $`git push origin master` 
