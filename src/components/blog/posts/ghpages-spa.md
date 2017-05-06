Single-Page Apps (SPAs) are all the rage right now.  They make it extremely easy to build feature-rich, highly performant web applications.  So you go ahead and build yourself a SPA, but one question remains.  Where do you host it?

This article is separated into two parts.  First, an outline of the different SPA hosting methods I have used, along with their pros and cons.  Second, my experience with hosting an SPA on GitHub Pages, including any trouble I have had.  I hope to help the readers make an informed decision regarding where to host their app, and if they do choose to use any of the methods I talk about, learn from my mistakes.

If you wish to focus on GitHub Pages, and skip the other methods, you can jump to section 4: GitHub Pages.

#### 1. Google App Engine

I didn't really get into web development until I started my current job.  Yes, I knew some HTML and CSS, and I had taken a university web dev class.  However, I wasn't all that interested until I started using those skills in the workplace.

At my job, all of our core applications are built using App Engine.  We use the Python version, with Jinja2 as our templating language, and Knockout.js for frontend code.  After a while I began to feel very comfortable developing in App Engine.  It was familiar, and I had gotten to the point where I could throw together a new application relatively quickly and easily.

Now, back to SPAs.  In the summer of 2016, I started working on [Roll Up Tracker](https://www.rolluptracker.com/).  Roll Up Tracker is an Angular2 web-app that allows you to track your wins and losses during the Roll Up The Rim To Win season at Tim Hortons.  Naturally, when I started the project I decided to host it on App Engine.  It might seem a bit odd to have to bootstrap an entire App Engine project just to serve some HTML and Javascript.  However, I justify it by the fact that the backend was written in Python, using Google Cloud Datastore as the database.  It made sense to keep the frontend hosted alongside that.

The implementation was pretty straightforward.

1. Add a couple rules for the frontend files, and a backend catch-all to your `app.yaml`
1. Create a wildcard route to a handler that just serves `index.html`
1. Create your API routes above your wildcard so they get matched first

```yaml
# app.yaml
- url: /assets/(.*)
  static_files: dist/assets/\1
  upload: dist/assets/.*

- url: /(.*\.(js|map))$
  static_files: dist/\1
  upload: dist/(.*\.(js|map))

- url: .*
  script: main.app
  secure: always
```

```python
class MainHandler(BaseHandler):

    def get(self, *args, **kwargs):
        context = {}
        self.response.write(template.render(os.path.join(TEMPLATE_DIR, 'index.html'), context))
```

That's about it!  Moving on to method number two...

#### 2. Google App Engine Flex

Oooooh "App Engine Flex"!  Sounds new and shiny and perfect and awesome.  It was, at first.

I had built a very small SPA using Vue.js called [Scrobblin' With Friends](https://scrobblin-friends.appspot.com/) as an attempt to bring Last.fm back into my life.  It's a simple app that allows you to enter your Last.fm username, and it shows you what all your friends are listening to.  It's a tiny app, with absolutely zero backend.  All it does is take in a username, hit an API every ~10 seconds, and display some data.  It definitely didn't make sense to bootstrap an App Engine project.

Then I discovered Flex, and found out they have a Node.js environment.  It was extremely easy to set up:

1. `app.yaml` with 2 lines:  `runtime: nodejs` and `env: flex`
1. `package.json` with a `start` script
1. A simple express server

```js
// server.js
var express = require('express');
var app = express();

const PORT = process.env.PORT || 8080;

app.get('/static/*', function(request, response){
     console.log('static file request : ' + request.params[0]);
     response.sendFile( __dirname + '/static/' + request.params[0]);
});

app.get('*', function(request, response) {
    response.sendFile(__dirname + '/index.html');
});

app.listen(PORT);
```

I finished the setup very quickly, deployed it, and life was good.  That is, until I got my bill for the month.

One of the big differences you notice between the Standard App Engine environment, and the Flex environment, is that your project never completely spins down.  On Standard, if nobody is hitting your site, you don't pay.  Even if you're getting a little bit of traffic, they're quite generous with the free tier.  Unbeknownst to me, "free tier" isn't a thing on Flex.  I racked up about $30 in a couple days before I noticed.

I could see Flex being a viable option if you have a project you were willing to pay big bucks to host.  This was not one of those projects.  So back to App Engine Standard I went.

#### 3. now

If you haven't heard of [now](https://zeit.co/now), go check it out now (heh).  It is seriously cool.  After installing it on your computer, just type `now` on your command line, and next thing you know, your application is on the web.

The only requirement is that you have a `package.json` with a `start` script.  `now` takes care of the rest.

As cool as the deployment process is, there were some things I didn't like about it.

1. The free tier gives you these janky URLs for your project, for example: coolproject-glkqdjsslm.now.sh.  I usually don't mind having strange URLs when I'm not paying for hosting, but the issue is that every time you deploy, you get a new URL.  You basically can't host anything that other people are going to use for free
1. Their cheapest paid tier, which gets you 1000 deploys/month (instead of 20), private codebases, and more is $14.99/mo.  Much more than I was willing to pay for silly side projects.

#### 4. GitHub Pages

Finally.  You sat through the excruciating minutia of my SPA hosting experiences.  We have finally arrived at the point of the article, which is to outline my experiences hosting an SPA on GitHub Pages.

It seems like the logical place to serve a SPA from, right?  You've got free hosting.  Deploys are a breeze.  Free SSL.  Plus your code is probably going to be kept on GitHub anyway!

I had recently decided that it was time to replace my personal website, which was previously a free Wordpress site that I put zero effort into (and I eventually password protected it because I was embarrassed by it).  My new site, which you are currently looking at, is written in Vue.js with no backend, so I could really choose any provider I wanted.  App Engine (both Standard and Flex) were overkill and/or too expensive, `now` was also unfeasible because of the reasons I stated above.  So here we are.  GitHub Pages is the chosen one.

It is not without it's faults.  Of course, there are a few quirks here and there that you might need to work around, but I haven't come across any blockers, and I'm sure you wouldn't either.  That said, if GitHub Pages were a paid service, some of the problems I encountered would have made me throw in the towel and switch to something else.  Since it's free, I was willing to put in the time to work around the issues.

I'd like to walk through how the whole thing works, including any "gotchas" that I encountered along the way.

### Part 1: What is GitHub Pages?

First off, [go here](https://pages.github.com/).  It has all the basics of Pages, and if you want more detailed documentation you can click "Pages Help" in the top-right corner.

There are two types of Pages.  The first is an Organization/Personal page.  This lives in a repo called `<YOUR_USERNAME>.github.io`, has a URL that matches the repo name, and is meant to be a general landing page for a person or organization (duh).  The code for the site lives in this project.

The other kind, a Project page, most commonly lives in the `gh-pages` branch, or in a `/docs` folder on `master`.  Then, you access the page from `<YOUR_USERNAME>.github.io/<PROJECT_NAME>`.  It allows you to keep all the code in one place.  Pretty handy, I think.

For the purposes of this post, I will be talking about Personal pages.

### Part 2: What should my repo look like?

This was the first problem I came across.  A personal site is served from the `master` branch.  That meant I couldn't just create a project using the Vue CLI, commit everything to master, and expect it to work.

After doing some research online, I settled on a plan.  I created a `sources` branch that houses the source files.  Then, on my computer, I built the project, initialized the generated `dist` directory as a git repo, set it to track `master`, then committed and pushed.  It worked pretty well, but it was largely manual.  Plus if I ever re-cloned the project or used another computer, I'd have to set up the dist repo again.  Annoying.

### Part 3: Automating Deployments

I was not about to settle for the process laid out above, so I figured I could devise a script that could do that for me, then just set up Travis CI to run it.  There are quite a few scripts floating around the internet that do exactly that.  I played with one for a while, tweaking it to work for my project, when I found out that Travis actually has a GitHub Pages deployment provider.  A couple commits later, I ended up with a pretty simple `.travis.yml` that took care of everything for me.

```yml
language: node_js
node_js:
  - "node"
script:
  - yarn run lint
  - yarn run build
deploy:
  provider: pages
  skip_cleanup: true
  github_token: $GITHUB_TOKEN
  local_dir: dist
  target_branch: master
  on:
    branch: sources
```

It basically just runs my lint and build, then if the branch is `sources`, force-push the `dist` directory to `master`.  For more info on this, check out the [Travis CI docs](https://docs.travis-ci.com/user/deployment/pages/).

### Part 4: Routing

It was very smooth sailing for a while.  I was developing locally very quickly using the webpack dev server, complete with hot module reloading.  When I wanted to deploy, I just pushed up my changes and in a minute or so they were live.  I had no problems setting up the Vue router at all.

As it turns out, when I was using the live site I was always starting from the root of the site.  I never tried to go directly to `/projects` or any other page.  That's when I hit a snag.  Trying to go directly to a route in my app resulted in a 404 page.

I definitely wasn't shocked when I discovered this.  It makes total sense, since GitHub doesn't know about the Vue app's routes.  All it knows is there is an `index.html` file in the repo that it can route to.

I attempted two different solutions to this problem.  Depending on your requirements, one may be more appropriate than the other.

#### Attempt 1: Hacking the 404 page.

I found another person online that was trying to solve the same problem as me.  You can see his blog post [here](http://www.backalleycoder.com/2016/05/13/sghpa-the-single-page-app-hack-for-github-pages/).  I'll admit it's quite a janky solution to the problem but it was also simple to implement.  Until GitHub comes up with a better solution, this is a pretty decent workaround.

This solution is based on the fact that you can use a custom 404 page on GH Pages.  All you have to do is add a `404.html` page to your repo.  Following the link above, I created `404.html` with the following lines in `<head>`:

```html
<script>
  sessionStorage.redirect = location.href;
</script>
<meta http-equiv="refresh" content="0;URL='/'"></meta>
```

Basically all this does is save the requested URL in your browser's storage, and boot you back to the root of the site.  Obviously, that's not enough.  If someone click a link to this blog post, I wouldn't want them being sent to the homepage.  The next step was just as simple.  I basically added another script to `index.html` to check for that saved URL, and redirect to it if it exists.

```html
<script>
  (function(){
    var redirect = sessionStorage.redirect;
    delete sessionStorage.redirect;
    if (redirect && redirect != location.href) {
      history.replaceState(null, null, redirect);
    }
  })();
</script>
```

This worked surprisingly well.  However, once I decided to implement Open Graph and Twitter meta tags in my blog posts, it fell apart due to the scrapers not following the redirects.  However, if you have a simple site that doesn't require meta tags, then this just might work for you.

#### Attempt 2: Pre-rendering a Static Site

There were two issues with the above router hack in terms of serving meta tags:

1. When Facebook, Slack, Twitter, or any other site with rich media cards would scan my links, they would land on the 404 page as expected.  However, they would not get redirected to the proper page by the method I shared above.  This is because GitHub Pages returned a 404 status code for the 404 page, which told the scrapers to give up.
1. Even if I didn't use the 404 page hack, my meta tags were generated on the fly in my Vue app.  Since the scraper downloads the page without executing any javascript, it would not find the meta tags.

So, the two obvious solutions are server-side rendering or pre-rending.  I didn't want to go down the path of server-side rendering, since I don't want to host a server for this website.  So, pre-rendering it is!

Since I'm using Webpack, I thought it made sense to add a webpack plugin to generate static HTML from my site.  I found a couple prebuilt plugins to do this, which both leveraged PhantomJS to render the site.  For some reason, I could not get it to render any actual content in my site.  I wasted a ton of time trying to get it working, with no success.

Frustrated with that, I decided to write my own plugin.  Thus, [webpack-static-site-generator](https://github.com/esalter-va/webpack-static-site-generator) was born (great name, I know).  With that plugin in place, every time I built my site for production I would get static HTML files for all of my pages.  I could still use the Webpack dev server when developing locally without building, which was a huge plus.

So, if you have any trouble hosting your SPA on any hosting provider, you might want to check out pre-rendering.

### Part 5: Custom Domain

If you've made it this far, you should be done with the "gotchas" of hosting a single-page app on GH Pages.  Setting up a custom domain was very easy.  There were just a couple steps:

1. Create a file called `CNAME` in your repo, which contains a single line with the domain you want to use (i.e. `www.evansalter.com`).  Make sure this file get commited to your master branch when the build runs.
1. Set up a `CNAME` record with your domain registrar that points to your `github.io` address (i.e. `esalter-va.github.io`)

There are some slightly different instructions on GitHub's support pages if you want to use an apex domain (no www), but it shouldn't be too much work.

There was one slight problem I discovered.  There is NO SSL support when you're using a custom domain with GH Pages.  I was a tad annoyed at this, but seeing as my site has no real reason to be on HTTPS, I got over it.

### Final Thoughts

I very excited about using GitHub Pages to host a single-page app when I first started this site.  However, it wasn't quite as magical as it seemed.  I faced many problems along the way.  Some of them had simple solutions, while others were a little more complicated.  However, I'm still happy with how it worked out in the end.  The simplicity of Pages is what makes it great, and I would recommend that everybody at least considers it for their next project.
