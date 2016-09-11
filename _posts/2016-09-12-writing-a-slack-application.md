---
layout: post
title: Writing a Slack Application
---

I managed to get my (first) Slack Application ([xqz.es](https://xqz.es))
published this week, this is the start-to-finish walkthrough

# Inception

It all started about six months ago, a friend and I had agreed to go 
to the gym together in the mornings before work.
On the first day we were already making excuses as to why we 
couldn't meet each morning, from the very serious to the downright ridiculous.

I jokingly said I should put up one of those simple websites that only show one line
of text, displaying our excuses. I slung together a bottle app 
(Easy start, familiar language and a framework to get familiar with).
I Bought a domain and ran the whole thing 
under the bottle dev server. It looked something like this:
![first prototype](https://arahayrabedian.github.io/images/2016-09-12/xqzes_first.png)

Ugly.

# Slash Commands

I had some spare time, I looked up how to write a
[slash command](https://api.slack.com/slash-commands), simple enough to
experiment with, so I put together one for a team with friends.

A simple bottle view that would pick a random excuse from an array and 
return one in the format Slack expects. No fancy checking of anything:

```python
@post('/xqz-moi')
def make_an_excuse():
    return {
        "response_type": "in_channel",
        "text": choice(EXCUSES),
    }
```

This **must** be served over HTTPS in order for Slack's systems to
even consider hitting your API. At first I thought "damn", but then
I remembered [Let's Encrypt](https://letsencrypt.org/), setting up
Let's Encrypt from knowing nothing to getting it working flawlessly 
was a one hour process. Highly recommended. Many online tutorials
so I won't get in to it.

Setting up the slash command itself was quite a breeze. There are 
very thorough examples on how this is done, but a quick rundown
using the snippet above as a basis. You must create a new 
[custom integration](https://slack.com/apps/build/custom-integration)
just for your team. Here you'll find "slash commands" option.

Pick the name of your command and hit the "Add Slash Command Integration"
button. This takes you to a configuration menu where they explain a few
things. I will only cover the important details.

For the `url` you must enter the address of the command you wrote 
above, for example, my case was originally `https://<domain>/xqz-moi`.
I highly recommend the `POST` version of this as this is **required** when
you convert from a slash command to a Slack application with an app command.
To pay attention to the token is *technically* optional 
(not sure how happy Slack would be about that.)
but highly recommended if you plan on expanding past just a command 
for your team (it ensures the privacy of teams[^1]). And really, it's
a simple string comparison. Do it.

The rest of the options are pretty self explanatory. When all is said an done
you should have something that looks like this:

![first slack prototype](https://arahayrabedian.github.io/images/2016-09-12/xqzes_slack_one.png)

'xqzmoi' (at the time) stayed like this for quite a while 
as life took over. Once I had some free time I tried to 
submit to the Slack application directory.

## Converting from Slash Commands to Slack Applications (App Commands)

Actually converting from a custom integration to an application with
an app command is fairly simple. There *are* some minor differences 
between having a slash command just for
your team and having an application. In particular, your app is no longer
for just one team (obviously), this means security considerations, writing
the application in a way others can easily grasp as well as some differences
that Slack itself brings up. For the technical details, Slack provides
a better overview than I can muster while describing 
[slash commands](https://api.slack.com/slash-commands)
(jump to the "App Commands" section).

When you create your own app, they allow you to copy over a slash command
from your currently signed in teams. This is highly recommended if you
developed your slash command already and don't feel like reconfiguring
it all again.

## Add To Slack (Implementing OAuth2)

I have always hated implementing OAuth and OAuth2. A fear ingrained from
the days of maintaining a bad implementation. It was time to overcome 
these fears in order to get my app to go from a custom slash command 
to something other people could install.

Slack provides an (html) snippet generator for your button on their
page documenting the [Add to Slack](https://api.slack.com/docs/slack-button)
button. The button is how teams will go about installing
your application. The snippet starts the OAuth2 process. I did a quick
search for ready-made python libraries and settled on `requests-oauthlib`.
The library (plus the way Slack handles OAuth) made it super easy to implement.
You only need one view (we'll call it the callback) that Slack hits for you
with all the details you need to pick up an access token from them, in
bottle this would looks something like:

```python
@route("/oauth2/callback/")
def callback():
    """ Step 3: Retrieving an access token.
    The user has been redirected back from the provider to your registered
    callback URL. With this redirection comes an authorization code included
    in the redirect URL. We will use that to obtain an access token.

    NOTE: your server name must be correctly configured in order for this to
    work, do this by adding the headers at your http layer, in particular:
    X_FORWARDED_HOST, X_FORWARDED_PROTO so that bottle can render the correct
    url and links for you.
    """
    oauth2session = OAuth2Session(settings.SLACK_OAUTH['client_id'],
                                  state=request.GET['state'])

    token = oauth2session.fetch_token(
        settings.SLACK_OAUTH['token_url'],
        client_secret=settings.SLACK_OAUTH['client_secret'],
        authorization_response=request.url
    )

    # TODO: store the token in some highly secure bunker.

    redirect('<insert_your_success_page_here>')
```

Where, `client_id`, `client_secret` and `token_url` are all provided to you
by Slack.

## Submitting your Application

So here I am thinking: "Great, everything's ready, time to get this 
on the Slack App Directory". I try and submit and they respond fairly 
quickly that I'm lacking several things (I'm paraphrasing here):

### 1 - Homepage is a little dull
Fair enough, it was the most basic of html and my potential users
would be seeing this page when they wanted to add the app.

I ask [some friends](https://xqz.es/acknowledgements/) to see if they'll toss 
in some CSS (since I have near-zero front-end capabilities), 
multiple step up to the challenge. I also ask for a logo
from friends and 30 seconds later I have a logo. Awesome.

### 2 - Github Issues not good enough for user support
Toss in a "Contact Us" page: WTForms + nocaptcha + email-it-to-me.

### 3 - Eliminate NSFW content by default
This was a tough one as different people have different definitions of NSFW,
after a few back and forths we agreed that removing one particular item was
good enough and would work on building a content rating and selection system
in the future if I really wanted to put in NSFW stuff.

## Done - What Now?
Well, xqz.es is finally on the 
[slack app directory](https://slack.com/apps/A270Z7C3Z-xqzes).
You can go ahead and install it and 
[give me feedback](https://xqz.es/contact/) :)

There aren't many excuses on there at the moment, but I'm hoping
more to flow in!

# Footnotes

[^1]: By checking the token (over an HTTPS connection) you can ensure that requests are legitimately sent by Slack, and not some malicious third party that wants the results of a slash command by pretending to be another team.
