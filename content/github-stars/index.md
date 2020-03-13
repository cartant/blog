---
title: "How to Find Projects Using GitHub Stars"
description: A look at how stars can be used to discover and choose projects
date: "2020-02-23T15:24:00+1000"
categories: []
keywords: []
cardImage: "./title-card.jpeg"
---

![Star](title.jpeg "Photo by Daniel Olah on Unsplash")

It's pretty common to refer to a GitHub project's star count. For example, at the moment, React has 144,162 stars, Angular has 58,093 and Vue has 157,325.

Those are some big numbers, but what do they mean? Would you choose a framework based on star counts? I wouldn't. However, I do think GitHub stars are useful and I use them every day. And that's what this post is about.

GitHub has some pseudo-social features that not everyone seems to use. Well, I'm pretty sure that some people don't, because several of the developers I've spoken with recently didn't know about or use these features.

It's possible to follow other people on GitHub: you visit their profile and click the follow button. They are then added to your following list and you're added to their followers list.

Once you have followed some people on Github, things get interesting — and useful — when you visit your dashboard. Your dashboard is what you see if you navigate to [github.com/](https://github.com/) — when you're logged in, of course.

The middle of the dashboard shows your 'Recent activity': your comments, issues, PRs, reviews, etc. Which is somewhat useful, but I deal with most of that stuff via email notifications. The interesting bit is below that: 'All activity'.

The 'All activity' section shows you some of the things that the people you are following have been up to. The section includes:

- repos they've created;
- repos they've made public;
- repos they've forked;
- repos they've pushed to — if you are watching the repo;
- repos they've starred; and
- people they've followed.

It's interesting to see people's activity — particularly if they've just created a new project — but the most useful inclusion, for me, are the repos that people have starred. It's a great way to find out about new packages. It's often that case the that people you've chosen to follow have similar interests and are solving the same kinds of problems that you are, so repos that they've decided to star are quite likely to be of use — or, at least, of interest — to you. Most of my package discoveries are made this way.

The relationship between followers and stars is presented in another part GitHub's user interface, too. When you click a repo's star count, it navigates to a page that gives you a paginated view of the repo's stargazers — the people who starred it. On that page, there are two tabs that separate said stargazers into the people you know — that is, the people you follow — and everyone else.

For me, this is really useful. When I'm trying to decide which package to use, I'll take a peek to see whether or not someone I know has starred its repo. If one package has no stars from people I know and another has several, it's likely that I'll choose the latter. There are other factors, of course, but this is useful information.

GitHub stars are also useful as bookmarks. It's often the case that I star something because it looks interesting and I anticipate its being useful in the future. Fortunately, the list of starred repos can be filtered, so when I'm looking for some CLI-related package that I can remember starring I can filter the list using ['cli'](https://github.com/cartant?tab=stars&utf8=%E2%9C%93&q=&q=cli) as the search term.

If you clicked the link in the previous paragraph, you'll have noticed that it's possible to filter someone else's list of starred repos. This can be useful for 'recommendations' and I used this recently when I was trying to find a React context-related package that I'd seen starred on the dashboard — it wasn't in my list of starred repos. I was pretty sure that [Tom Crockett](https://github.com/pelotom) had starred it, so I filtered his list using ['context'](https://github.com/pelotom?tab=stars&utf8=%E2%9C%93&q=&q=context) as the search term and found what I was looking for: [`react-tracked`](https://github.com/dai-shi/react-tracked).

When combined with the features outlined above, I think GitHub stars are useful for discovering and choosing packages. I just don't pay much attention to the total star counts.
