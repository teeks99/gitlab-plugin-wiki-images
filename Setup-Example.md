[This can also be found on Medium (with slightly better formatting).](https://medium.com/@teeks99/continuous-integration-with-jenkins-and-gitlab-fa770c62e88a#.c4j4to4ys) 

I recently got tasked with setting up a new Jenkins box within my organization, and having it work with our GitLab hosted Git repositories. We had previously had some rudimentary interaction between these two sides, with webhooks from our GitLab repo calling our Jenkins instance, using the GitLab Webhook Plugin. This worked alright, based on the branch specification in the Jenkins jobs it would perform the build/test when changes were made in the repo.

The downside to this method is that there isn’t any feedback in GitLab. I’ve recently worked on a project for my masters where we had a GitHub repo working with CircleCI, and with every commit to GitHub our build/test would run and we would get nice graphical statuses in GitHub. Below shows the nice status check:

![](https://cdn-images-1.medium.com/max/800/1*t7kBszjsNVDxvvrGlvOBQA.png)

It would sure be nice if I could get similar integration between GitLab and Jenkins. When I looked around at GitLab’s documentation, it mentioned that it supported connections to Jenkins, but only in the Enterprise Edition, which isn’t coming to our organization any time soon…bummer. It also mentioned that the Community Edition does work with GitLab’s own CI runner, but we have a bunch of other things that only run in Jenkins (we still have clearcase projects around here), so we couldn’t just move to that.

Luckily, while I was looking around for the Jenkins GitLab Webhook Plugin that we had used previously, I stumbled across the Jenkins GitLab Plugin. At first I thought it might just be another plugin that works similar to the webhook plugin, but after digging a bit deeper into the README on its GitHub Page, I realized that this was performing a similar service to what I was looking for!

To start the process, I installed Jenkins 2.7.2 as a service on windows (from the .msi installer), with the default plugins (includes the Git Plugin). I then installed the GitLab Plugin (version 1.4.0). While I’m naming versions, we’re currently running GitLab CE 8.11.2. The machine has Git 2.9.2 installed on it.

I next created a SSH key pair on the machine, which put the keys in C:\Users\MY_USER\.ssh\ (id_rsa for the private key and id_rsa.pub for the public key). I added the public key as a deploy key to my GitLab project(s). I then added my private key as a Jenkins Credential.

![](https://cdn-images-1.medium.com/max/800/1*vjVU8I33ca2Z60htG8Tvag.png)

Enter the credential manager

![](https://cdn-images-1.medium.com/max/800/1*-u7TUzPNeA77becht4OBQQ.png)

Go into system credentials

![](https://cdn-images-1.medium.com/max/800/1*z7eX-dPD_vCDQrSNxx3Kcg.png)

Add a new credential

![](https://cdn-images-1.medium.com/max/800/1*oBzdGpA4ekNCB0pCddMEnw.png)

We need a SSH private key credential. I’m not sure the username matters, but I set it to “git” as that is what the login for gitlab uses. I then pointed it to my *private* key on the machine and gave it a description.

![](https://cdn-images-1.medium.com/max/800/1*vPyus67JTkJMfHQKWpfA2g.png)

Now the key shows up in the system.

Once I had setup the ssh private key, which allows Jenkins to connect to the git host over ssh (i.e. git clone git@server:user/repo.git), I needed to setup API Access for Jenkins to get metadata from GitLab.

![](https://cdn-images-1.medium.com/max/800/1*XIv_9ZhN1Z5w3iRPRR7NPQ.png)

In GitLab go to your user profile.

![](https://cdn-images-1.medium.com/max/800/1*dDwVPvBGr1ndWzyoYRo9rA.png)

Select “Access Tokens”.

![](https://cdn-images-1.medium.com/max/800/1*tnw_1o1uW4WIHwiAMUfCpg.png)

Now make a new token, I gave it a descriptive name and made it expire at the end of the decade (I haven’t investigated the pros/cons of expiration date length yet).

![](https://cdn-images-1.medium.com/max/800/1*dUqHRULTyO_oll4ho8pn3Q.png)

You are then presented with a screen showing the token. Copy it now, as it won’t be accessible again! You can always create a new one if you mess it up though. (FYI I have revoked the token shown above)

![](https://cdn-images-1.medium.com/max/800/1*9rVAy2J90-Itn4aG89T8wA.png)

Back in Jenkins’s System credentials add a new one of the type GitLab API token. Paste in the API token shown in the last step.

![](https://cdn-images-1.medium.com/max/800/1*ynY5b_mPtqzkgV87M0Hgaw.png)

Now the API link has been added.

Once the API Access has been setup, we can configure the connection between Jenkins and GitLab. This happens in the Mange Jenkins -> Configure System menu.

![](https://cdn-images-1.medium.com/max/800/1*6OJOhTEDmedsjc7rvVm_ww.png)

Give it a name, the host where GitLab’s API lives, and the credential for the token created in the last step.

Now that our connection between Jenkins and GitLab is setup, we need to create a job. I’ve only worked with freestyle jobs, so if you’re doing a pipeline you’ll be on your own from here on out.

![](https://cdn-images-1.medium.com/max/800/1*8-SkRxyksh4UrFYHT2z3zw.png)

You’ll see that the connection you just made is automatically selected.

![](https://cdn-images-1.medium.com/max/800/1*bDGBSXIOLeRndZCC-xTaZw.png)

In the source code management section, enter the ssh URL of your repository. Select the ssh key generated above. Then we need to do the unique items.

We need to specify the name as “origin”, which will be used by the other sections. For the Refspec we need to input:

    +refs/heads/*:refs/remotes/origin/* +refs/merge-requests/*/head:refs/remotes/origin/merge-requests/*

I haven’t completely figured out how this works, but it seems to be correct for my use cases.

For the Branch Specifier we need `origin/${gitlabSourceBranch}` which will be filled in based on the web hook we’ll be setting up next.

![](https://cdn-images-1.medium.com/max/800/1*4uE3zKuv1GY1E_gkNww_9g.png)

For the trigger we want to set it up for changes going in to GitLab.

![](https://cdn-images-1.medium.com/max/800/1*oVylETyVct5ZmV204PE4DA.png)

Finally, in the post build step we need to “Publish build status to GitLab commit”. This enables the feedback and gives us pretty indicators in GitLab.

![](https://cdn-images-1.medium.com/max/800/1*CCAWHLr_dzwolf4E8_QeqA.png)

That’s everything from the Jenkins side. Now we need to setup the GitLab side to trigger the builds. We’ll do this with webhooks, and we’ll need one per Jenkins job.

![](https://cdn-images-1.medium.com/max/800/1*DH4zd4AtT__8Sf1Wj2fVpg.png)

From the GitLab project you want to build, select the Webhooks option from the settings menu on the right.

![](https://cdn-images-1.medium.com/max/800/1*iLyQFcXRvlHwOVAS9_rYkQ.png)

You need to enter the URL of the jenkins server. The path is “project/JOB_NAME”. Select Push events and Merge Request Events.

![](https://cdn-images-1.medium.com/max/800/1*bC4lcClRT66Gg-kK5OoVwg.png)

Now everything should be set! If you push a commit to the repository, you should see the Jenkins job start running. Once the job completes, you should see the status next to the commit in GitLab.

![](https://cdn-images-1.medium.com/max/800/1*6Aah2g3BYL-janiIi3vEAw.png)

Here’s a few commits with their build result status.

![](https://cdn-images-1.medium.com/max/800/1*rmZGUCCxglZ5JYCf2WwKnA.png)
Clicking on the status icon shows some info about the build. Clicking on that Build ID (#6253) will take you to the Jenkins page for that build.

This setup works great for individual commits that are pushed into the repo. However, it also works fine for handling merge requests (despite some of the wording on the Jenkins GitLab Plugin’s readme page). I was able to have merge requests from separate branches in the same repo as well as from forked repos trigger builds and show the resulting status.

![](https://cdn-images-1.medium.com/max/800/1*Z4SMcLBMM0ykXNzetktKCw.png)

Merge request while the build is in progress.

![](https://cdn-images-1.medium.com/max/800/1*gIsde2yJSaJ8fiSz9Ok9NQ.png)
A merge request with the completed build. The View details link will take you to the same build status page as above, and on to the Jenkins page for that build.

All in all, I’ve been very happy with the use of the Jenkins GitLab plugin to support CI between Jenkins and GitLab. It was definitely an unexpected bonus after reading about how you need the EE of GitLab to hook to Jenkins, apparently that isn’t completely true.

The one issue that I have with it is that the Jenkins jobs are hooked too closely to the GitLab integration. For instance, I can’t just click the Build Now button on a Jenkins job using the integration, as it can’t find the origin/${gitLabSourceBranch} in the repo, since that variable is only set when a webhook request comes in. It would be nice if I could try to re-build a failed build (some of our projects are, sadly, too complex for deterministic testing behavior), but that isn’t in the cards. I’ve made separate jobs for running manual (or scheduled builds), but their status doesn’t show up back in GitLab.