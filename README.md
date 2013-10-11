Overview (v1.1)
===============
This directory stores FUN's theming files for its edX instance.
It is based on the Stanford theme. We're storing the stuff here 
and then pulling it in to our instance when we deploy.

Installation
============

Installation with ansible
-------------------------

lms/envs/common.py  also contains the variable 'USE_CUSTOM_THEME' and the function enable_theme, see its docstring
rakelib/assets.rake contains THEME_NAME = ENV_TOKENS['THEME_NAME']
One can add "THEME_NAME":"stanford" to env.json, still does not seem to make a difference


First, you need to surcharge the theme-related variables from 
`playbooks/roles/edxapp/vars/main.yml`:

```
edxapp_theme_name: 'fun'
edxapp_theme_source_repo: 'https://github.com/FUNMOOC/edx-theme.git'
edxapp_theme_version: 'mooclab/funisation'
```

Also, unless this was changed since the last time I installed a theme, you also need to
set the `THEME_NAME` like this, still in your ansible playbooks (the files
`/opt/wwc/*.json` are handled by ansible, so you shouldn't alter them directly):

```
generic_env_config: &edxapp_generic_env
    ...
    'THEME_NAME': 'fun'
    'MKTG_URL_LINK_MAP':
      "ABOUT": "about"
      "HELP": "help"
      "HONOR": "honor"
      "HOW-IT-WORKS": "how-it-works"
      "TOS": "tos"
      "FAQ": null
      "PRIVACY": null,
      "CONTACT": null
```

This will make ansible add the `THEME_NAME` variable  to the env tokens in `/opt/wwc/lms*.env.json`
(there can be several variants of the LMS service on a single host, and thus different
files).

This will also ensure that the new 'FAQ' page doesn't add a view requirement which isn't
handled by the theme (to remove once this is fixed upstream).

When playing the playbook with these variables, the theme repository is fetched and 
the configuration variables get the code from `edx-platform` to load the
templates and static files from the theme directory.

Note: this explanation is adapted from 
https://github.com/pdehaye/antoviaque-questions/blob/master/theming.utf8


Manual installation
-------------------

First, go one level up from the directory containing `edx-platform` on your installation (on 
production instances, it's usually `/opt/wwc` with `edx-platform` located at `/opt/wwc/edx-platform`).

```
$ cd /opt/wwc
```

Create a directory for themes and get the theme source repository in a folder nammed `fun`:

```
$ mkdir /opt/wwc/themes
$ cd themes
$ git clone git@github.com:FUNMOOC/edx-theme.git fun
```

Then change to the branch you would like to use:

```
$ cd /opt/wwc/themes/fun
$ git checkout xxxx
```

Probably the most obnoxious part of themes is that both Django and Rake need to know:

* whether or not a theme is enabled
* what the theme's name is (since that tells them where to look for templates, static files, etc.)

As is done in production environments, we use a JSON file located in the ENV_ROOT (the parent dir 
of your repo dir) to set the theme settings. The JSON file in dev environments is simply called 
env.json; in production, it's named slightly differently. If you want to turn a theme on, then 
you must have this file (or Rake won't invoke Mako and set up Sass properly), and if you want to 
disable the theme, then you either cannot have this file (or else Rake will invoke Mako and set 
up Sass to load the theme's Sass).
 
Here are the env.json with the proper values to enable the `fun` theme:

```
{
  "PLATFORM_NAME": "FUN",
  "SITE_NAME": "france-universite-numerique-mooc.fr",
  "DEFAULT_FROM_EMAIL": "inscription@france-universite-numerique-mooc.fr",
  "DEFAULT_FEEDBACK_EMAIL": "feedback@france-universite-numerique-mooc.fr",
  "DEFAULT_BULK_FROM_EMAIL": "cours@france-universite-numerique-mooc.fr",
  "SERVER_EMAIL": "dev@france-universite-numerique-mooc.fr",
  "TECH_SUPPORT_EMAIL": "helpdesk@france-universite-numerique-mooc.fr",
  "CONTACT_EMAIL": "contact@france-universite-numerique-mooc.fr",
  "BUGS_EMAIL": "bugs@france-universite-numerique-mooc.fr",
  "PAYMENT_SUPPORT_EMAIL": "paiements@france-universite-numerique-mooc.fr",
  "ADMINS": ["dev@france-universite-numerique-mooc.fr"],
  "THEME_NAME": "fun",
  "MKTG_URL_LINK_MAP": {
    "ABOUT": "about",
    "HELP": "help",
    "HONOR": "honor",
    "HOW-IT-WORKS": "how-it-works",
    "TOS": "tos",
    "FAQ": null,
    "PRIVACY": null,
    "CONTACT": null
  }
}
```

The THEME_NAME setting is the important one, as it's the themes/<theme-name> directory that Rake 
looks for when compiling assets (Django will also reference it too). The others are standard 
overrides that you could just as well specify in your custom lms/envs/<settings>.py file. All of 
the other settings, however, with the exception of SITE_NAME, are new in this PR.

Then you need to use Django settings similar to what is in lms/envs/aws.py for loading `ENV_TOKENS`:

```
with open(ENV_ROOT / "env.json") as env_file:
    ENV_TOKENS = json.load(env_file)

PLATFORM_NAME = ENV_TOKENS['PLATFORM_NAME']
SITE_NAME = ENV_TOKENS['SITE_NAME']

#Theme overrides
THEME_NAME = ENV_TOKENS.get('THEME_NAME', None)
if not THEME_NAME is None:
    enable_theme(THEME_NAME)
    FAVICON_PATH = 'themes/%s/images/favicon.ico' % THEME_NAME

DEFAULT_FROM_EMAIL = ENV_TOKENS.get('DEFAULT_FROM_EMAIL', DEFAULT_FROM_EMAIL)
DEFAULT_FEEDBACK_EMAIL = ENV_TOKENS.get('DEFAULT_FEEDBACK_EMAIL', DEFAULT_FEEDBACK_EMAIL)
DEFAULT_BULK_FROM_EMAIL = ENV_TOKENS.get('DEFAULT_BULK_FROM_EMAIL', DEFAULT_BULK_FROM_EMAIL)
SERVER_EMAIL = ENV_TOKENS.get('SERVER_EMAIL', SERVER_EMAIL)
TECH_SUPPORT_EMAIL = ENV_TOKENS.get('TECH_SUPPORT_EMAIL', TECH_SUPPORT_EMAIL)
CONTACT_EMAIL = ENV_TOKENS.get('CONTACT_EMAIL', CONTACT_EMAIL)
BUGS_EMAIL = ENV_TOKENS.get('BUGS_EMAIL', BUGS_EMAIL)
PAYMENT_SUPPORT_EMAIL = ENV_TOKENS.get('PAYMENT_SUPPORT_EMAIL', PAYMENT_SUPPORT_EMAIL)
ADMINS = ENV_TOKENS.get('ADMINS', ADMINS)
MANAGERS = ADMINS

# Marketing link overrides
for key, value in ENV_TOKENS.get('MKTG_URL_LINK_MAP', {}).items():
    MKTG_URL_LINK_MAP[key] = value
```

Note: A portion of these instructions come from
https://github.com/edx/edx-platform/pull/57 which gives more 
details about the internals of themes activation, if needed.

