---
title: "Setting Up Gitea Part2"
summary:  "Setting up Gitea part2, making users and organizations"
date: 2025-08-23
tags: ["deployment", "git", "source","gitea"]
draft: false
---

Now that Gitea is running we want to connect it to our identity management system.  To this we are first going to create the OIDC client in PocketID. 

Login with an admin account in your PocketID and click on OIDC clients.


![OIDC Client](<images/Pasted image 20250723070710.png>)


Then click OIDC client.

![Add Client](<images/Pasted image 20250723070734.png>)

Give it a name and callback URL.  The callback URL should be your domain with the same structure as below.

It will look something like this.

![Callback URL](<images/Pasted image 20250723071052.png>)

This will generate a Client ID and Client Secret

{{< alert >}}
**Warning**
This will be the only time you will see the client secret, so store someplace secure for now.  
{{< /alert >}}

![Secret Key](<images/Pasted image 20250723071217.png>)


Now we have to configure our Gitea instance to use these keys.  Start by logging in as the admin user in Gitea.  

Under your profile you can find Site Administration

![Settings](<images/Pasted image 20250723081100.png>)
From there, go to Identity & Access
![Identity](<images/Pasted image 20250723081224.png>)
You then want to choose Authentication Sources and on the far right is the option for "Adding Authentication Source"

![Source](<images/Pasted image 20250723081346.png>)


Under Authentication name put PocketID

Under OAuth2 Provider choose OpenID Connect


The Client ID and Client Secret come from PocketID.

Under the "OpenID Connect Auto Discovery URL you want something Similar to this"

`https://identity.dev.th3redc0rner.com/.well-known/openid-configuration`

Your configuration should look something like this.

![Keys](<images/Pasted image 20250723081846.png>)

Then click Add Authentication Source on the very bottom.

Logout of your admin account, then click on Sign In Again.

If all went well you should be presented with a PocketID option.

![Sign In](<images/Pasted image 20250723082041.png>)
Now try signing in with your account and should let you use a passkey to authenticate.

Before creating our organizations I created two more users in my identity manager.

 ![Users List](<images/Pasted image 20250723220511.png>)

Bob will be our organization admin

Sam will be a regular operator.  


Now Let's login as Bob in our Gitea instance.

![Bob Login](<images/Pasted image 20250723220649.png>)
First click on organizations after logging in.

![Organizations](<images/Pasted image 20250723220955.png>)

Then click on the **plus** symbol to create a new organization

![New Org](<images/Pasted image 20250723221102.png>)

Now I am going to create an organization called RedCorner.  I made this organization limited to only authenticated users but overall its internal so that settings is up to how you want to create the repositories.

So the first thing I did was create 3 repositories in our organization:

- k8sclusters -- will be used for flux to manage our cluster and main infrastructure, only admins will have access to this repository. 
- Excercises -- this will specific to our c2 infrastructure and operators will be able to modify this repository
- infrascripts -- this will hold the ansible and k8s initial deployments setup, misc scripts etc



Once the Organization is created we can setup our groups.  The first group I am going to make is RepoAdmins and am going to give that team Administrator Access to all repos.


![RepoAdmins](<images/Pasted image 20250723221423.png>)

I then made the k8sclusters and Exercises private so only people in the organization can see this repo.  

![Private Repo](<images/Pasted image 20250723221838.png>)



When we're done the organization will have 3 repositories:

![List of Repos](<images/Pasted image 20250723222323.png>)

Now to set access rights we would go to teams.

The first team you want to create is the RepoAdmins:

This team will have full admin access to the organizations repositories.  

Click New Team:


![New Team](<images/Pasted image 20250723222651.png>)


Operators will have admin access to specific repositories:
![](images/Pasted image 20250723222848.png>)
Once you have done that click on the Operators team and click repositories:

Search for exercises:

![](<images/Pasted image 20250723223109.png>)
Then add it to this team.

Now to get Sam into the group, log in as him. Gitea won't recognize a user until after their first login.

Once Sam is there click on members and search for him and Bob.  Add them both to this repo:

![](<images/Pasted image 20250723223233.png>)


You should now see both members listed in the group:

![](<images/Pasted image 20250723223304.png>)

Under RepoAdmins add Bob only:

![](<images/Pasted image 20250723223339.png>)

Now our organizations are setup we can now go back to our infrastructure scripts on our jumpehost and push them to to the infrascripts repo.


On our original jumphost we can retieve our ssh keys.  Now this was more for demo purposes to use Bob, i added myself to repoadmins also so my name is on my commits and not Bob.

![](<images/Pasted image 20250723224927.png>)


The next step is to add these keys to a user -- in this case, I'll add them to my own account.

To setup the ssh keys you click on ones profile image and click settings

![](<images/Pasted image 20250724221420.png>)
Then you would choose SSH/GPG Keys

![](<images/Pasted image 20250724221456.png>)

Then you can click Add Key.  The keys we add are always the public key and the private keys are what we have on our jumpbox.

The first key we will add is the

`admin1_git_ed25519_key.pub`

One should end up with something like this

![](<images/Pasted image 20250724221728.png>)

We can add the flux key the same way
![](<images/Pasted image 20250724221927.png>)

Let's verify that our keys are working.  

You should be able to do something similar to this:

```css
ssh git@gitssh.dev.th3redc0rner.com -i ~/.ssh/admin1_git_ed25519_key
```

If our key is working you will get the response similar to this:
```
Hi there, skragen! You've successfully authenticated with the key named admin1_git_key, but Gitea does not provide shell access.
```

To make things more convenient and cleaner I also added it to my ssh config file

```css
vi ~/.ssh/config
```


The entry looks like this

```css
Host gitssh.dev.th3redc0rner.com
  User git
  Port 22
  IdentityFile ~/.ssh/admin1_git_ed25519_key
```
So now that our keys are added to Gitea we can push our infrastructure repo.

Let's go to the jumpbox.

Go to our infrascripts directory.  Perform the following commands.

```
git init
git checkout -b main
git add .
git commit -m "First Commit"
git config --global user.email <your email>
git config --global user.name <your username>
git remote add origin git@gitssh.dev.th3redc0rner.com:RedCorner/infrascripts.git



```

If you go back to Gitea and open your infrascripts repository you should see it has updated with your current scripts

It should look similar to this 
![](<images/Pasted image 20250724223646.png>)
I have made a public version of this repo.  You can find it at **https://github.com/TheR3dC0rner/infrascripts**






