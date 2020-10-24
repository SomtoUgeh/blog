## Get Production-ready with Strapi + Gatsby

It is a lovely Saturday afternoon and you are looking for new tech to try out; maybe for a side gig or something. You've heard so much about Gatsby and you definitely need to have it under your Frontend belt to excel in this streetsss. You heard over to [Gatsby](https://www.gatsbyjs.com/) and set it up. You look around and it fairly similar to the [create-react-app](https://reactjs.org/docs/create-a-new-react-app.html) you've been using for a while. However, you would like to build something dynamic, without really setting up your own backend. 

You reach out for a CMS (Content Management Systems) to solve your worries. After some googling, you find out about [Strapi](https://strapi.io/); a nice, ***FREE***, headless CMS. You go through the docs and find a tutorial on [building a blog with Gatsby](https://strapi.io/blog/build-a-static-blog-with-gatsby-and-strapi) [[code](https://github.com/strapi/strapi-starter-gatsby-blog-v2)]. By the way, if you ever want to really get the difference between CMS and Headless CMS, please check out this [thread by Chidi](https://twitter.com/ChidiWilliams__/status/1259932965796294656).

You follow through the tutorial and you are ready to deploy the applications; Gatsby frontend and Strapi Backend. This article is written to save you hours of figuring out how to go about that. 

## Deploy Backend

Let's get somethings out of the way. I am assuming:

- You are using the `--quickstart` option to setup strapi
- [Heroku account](https://signup.heroku.com/) for hosting strapi instance
- A [Cloudinary account](https://cloudinary.com/users/register/free) for saving images for free

If you would like to use another service like Amazon AWS EC2, please follow this [documentation](https://strapi.io/documentation/v3.x/deployment/amazon-aws.html#amazon-aws) [[docs](https://strapi.io/documentation/v3.x/getting-started/introduction.html)]

The first thing you need to do is make sure you are signed in to your Heroku account. You can do this via the Heroku CLI. 

```bash
# Download the installer if you don't have it
brew tap heroku/brew && brew install heroku

# Login to heroku
heroku login

# Follow the instructions and return to your command line
```

The next step is to create a new heroku project

```bash
# path: ./my-project
heroku create

# You can use heroku create custom-project-name, 
# to have Heroku create a custom-project-name.heroku.com URL. 
# Otherwise, Heroku will automatically generate a random project name 
# (and URL) for you.

# Note: If you have a Heroku project app already created. 
# You would use the following step to initialise your local project folder:
git remote rm heroku
heroku git:remote -a your-heroku-app-name
```

Good, we have Heroku setup and our application will be ready in a few minutes 😜. 

If you are very impatient like me and you try to deploy your app at this point. You would run into some trouble, you would notice that after every deployment, your *database* is swiped clean and you have to start again*.* 

It's wild, it took me some time to figure out because I did not read the docs properly. 

> When you use `--quickstart` to create a Strapi project, a SQLite database is used which is not compatible with Heroku. Therefore, another database option must be chosen. 

We will be using PostgreSQL here. If you would like to use a service like MongoDB, check out the docs [here](https://strapi.io/documentation/v3.x/deployment/heroku.html#_7-heroku-database-set-up)

Let's setup PostgreSQL to work with Heroku:

- Install the Heroku Postgres add-on for Postgres

```bash
# path: ./my-project/
heroku addons:create heroku-postgresql:hobby-dev
```

- Retrieve database credential

```bash
heroku config

# This should print something like this:
# DATABASE_URL: postgres://ebitxebvixeeqd:dc59b16dedb3a1eef84d4999sb4baf@ec2-50-37-231-192.compute-2.amazonaws.com:5432/d516fp1u21ph7b.

# This url is read like so: 
# *postgres:// USERNAME : PASSWORD @ HOST : PORT : DATABASE_NAME*
```

- Setup environment variables

```bash
heroku config:set DATABASE_USERNAME=ebitxebvixeeqd
heroku config:set DATABASE_PASSWORD=dc59b16dedb3a1eef84d4999a0be041bd419c474cd4a0973efc7c9339afb4baf
heroku config:set DATABASE_HOST=ec2-50-37-231-192.compute-2.amazonaws.com
heroku config:set DATABASE_PORT=5432
heroku config:set DATABASE_NAME=d516fp1u21ph7b

# replace with actual values
```

- Update your database configuration.

```bash
# path: ./my-project/backend/config
mkdir env && mkdir env/production
cd env/production && touch database.js

# path: ./my-project/backend/config/env/production/database.js
module.exports = ({ env }) => ({
  defaultConnection: "default",
  connections: {
    default: {
      connector: "bookshelf",
      settings: {
        client: "postgres",
        host: env("DATABASE_HOST", "127.0.0.1"),
        port: env.int("DATABASE_PORT", 27017),
        database: env("DATABASE_NAME", "strapi"),
        username: env("DATABASE_USERNAME", ""),
        password: env("DATABASE_PASSWORD", ""),
      },
      options: {
        ssl: false,
      },
    },
  },
}); 
```

- Install the `pg` package

```bash
yarn add pg
```

Great! Now we have a database setup but there is a big disclaimer: *Due to Heroku's filesystem you will need to use an upload provider such as AWS S3, Cloudinary, or Rackspace.* Unlike with project updates on Heroku, the file system doesn't support local uploading of files as they will be wiped when Heroku "Cycles" the dyno—this happens any time you redeploy or during their regular restart which can happen every few hours or every day.

Remember the Cloudinary account we created earlier? We have to connect it to Heroku. 

```bash
# Get information from your cloudinary console
heroku config:set CLOUDINARY_CLOUD_NAME='CLOUDINARY_CLOUD_NAME'
heroku config:set CLOUDINARY_API_KEY='CLOUDINARY_API_KEY'
heroku config:set CLOUDINARY_API_SECRET='CLOUDINARY_API_SECRET'

# replace with actual values
```

- We will need to install the [cloudinary strapi provider](https://www.npmjs.com/package/strapi-provider-upload-cloudinary)—[[docs](https://strapi.io/documentation/v3.x/plugins/upload.html#install-providers)][[other providers](https://www.npmjs.com/search?q=strapi-provider-upload-&page=0&perPage=20)]

```bash
# path: ./my-project/
yarn add strapi-provider-upload-cloudinary
```

- After the installation, we need to update our upload configuration

```bash
# path: ./my-project/backend/extensions
cd extensions
mkdir upload && mkdir upload/config
cd upload/config && touch settings.js

# path: ./my-project/backend/extensions/upload/config/settings.js
if (process.env.NODE_ENV === "production") {
  module.exports = {
    provider: "cloudinary",
    providerOptions: {
      cloud_name: process.env.CLOUDINARY_CLOUD_NAME,
      api_key: process.env.CLOUDINARY_API_KEY,
      api_secret: process.env.CLOUDINARY_API_SECRET,
    },
  };
} else {
  module.exports = {
    provider: "local",
    providerOptions: {},
  };
}
```

- Commit your changes and deploy to heroku

```bash
git add .
git commit -m "Production ready"
git push heroku master && heroku open

# Great, now we are all set up on the backend 🥳 🎉 🎊
```

## Deploy Frontend

This article will focus on [Vercel](https://vercel.com/). It really easy and fun to setup.

To deploy the Gatsby, you'll need:

- [A Vercel account](https://vercel.com/dashboard) for free
- Wait for your Heroku instance to be up and running before deploying your Gatsby Blog
- Vercel will ask you the root directory of the project to deploy which is **`frontend`**

![https://raw.githubusercontent.com/strapi/strapi-starter-gatsby-blog-v2/master/medias/vercel-deploy-step-1.png](https://raw.githubusercontent.com/strapi/strapi-starter-gatsby-blog-v2/master/medias/vercel-deploy-step-1.png)

- Paste the URL of your running Strapi instance on Heroku without the trailing slash in the `API_URL` option in environment variables and **DEPLOY**. 

> This assumes you are using `API_URL` as an env value in `gatsby-config`

![https://raw.githubusercontent.com/strapi/strapi-starter-gatsby-blog-v2/master/medias/vercel-deploy-step-2.png](https://raw.githubusercontent.com/strapi/strapi-starter-gatsby-blog-v2/master/medias/vercel-deploy-step-2.png)

- Automatic build on Vercel

We're using Gatsby which is a static site generator (SSG). This means we need to trigger new builds when the content changes in Strapi. We'll use webhooks to do this automatically.

We first need to create a Deploy Hook in Vercel. In your project's settings, go to the end of the Git Integration tab. Name your hook however you want, but make sure you link it to your master branch.

![https://raw.githubusercontent.com/strapi/strapi-starter-gatsby-blog-v2/master/medias/vercel-deploy-hook.png](https://raw.githubusercontent.com/strapi/strapi-starter-gatsby-blog-v2/master/medias/vercel-deploy-hook.png)

Then copy the generated URL and open your Strapi admin in production. In the settings tab, open Webhooks and paste the hook URL. Make sure you check all events to trigger build after every change.

![https://raw.githubusercontent.com/strapi/strapi-starter-gatsby-blog-v2/master/medias/webhook-vercel.png](https://raw.githubusercontent.com/strapi/strapi-starter-gatsby-blog-v2/master/medias/webhook-vercel.png)

![https://raw.githubusercontent.com/strapi/strapi-starter-gatsby-blog-v2/master/medias/webhooks.png](https://raw.githubusercontent.com/strapi/strapi-starter-gatsby-blog-v2/master/medias/webhooks.png)

Now every time we make a change in Strapi (our backend), Vercel creates a new build! (our frontend)

Thank you for getting to this point! I wanted to document this process because I spent a full day trying to figure out some parts of the deployment process. Also please check out this proper post from the strapi team—[[docs](https://github.com/strapi/strapi-starter-gatsby-blog-v2)]

If you enjoyed the article, consider sharing it so more people can benefit from it! Also, feel free to [@me](https://twitter.com/somtougeh) on Twitter with your opinions.

### Gracias 🙏🏾