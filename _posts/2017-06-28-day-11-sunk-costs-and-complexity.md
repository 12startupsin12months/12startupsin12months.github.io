---
layout: post
title: "Day 11 - Sunk costs and complexity"
date:   2017-06-27
published: true
categories:
  - Complexity
  - Sunk Costs
---

The [Sunk Cost Fallacy](https://en.wikipedia.org/wiki/Sunk_cost) is an essential life lesson to learn early on,
especially for software developers.

I spent most of my time building a feature which I thought was the bees knees. However, as the feature progressed and I reached the end, I wasn't so sure about its usefulness.
This feature would allow users to host websites under subdomains with their usernames. So, If my username was '12s12m', my websites would be published under:
http://awesome-website.12s12m.sprymesh.com . One the face of it this looks like a good idea. Everyone loves their names and usernames :). However, now after adding the username
I had to make sure that the site's subdomain was setup properly on creation and updated properly when it was changed. I also had to add a username field to the users table
and since I create users from Dropbox automatically, I had to come up with a way to create unique usernames just by using the user's email.

At this point I started looking at the existing providers who could have done this but haven't gone this route. S3, Cloudfront, Fastly, GitHub Pages. Almost everyone provides websites
under a single word. So, at the end of the day I swallowed my pride and tore down all the new code.

There is another lesson that I had learnt earlier which I had forgotten. Make sure a single git commit only adds files relevant to a specific feature. Even if you've built 2 separate features
in the current working directory. Add the relevant files and create multiple commits. One for each change. This way, if you have to undo a commit. You can let git do it by using:

```
git revert commit-id
```

Anwyay, I am happy that at the end of the day. I don't have to support a cumbersome feature which doesn't add much value.

I'd love for you guys to try out https://sprymesh.com/ which I build it out :) Please share your feedback :)
