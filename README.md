# Spotify oAuth Proxy

[![Deploy](https://www.herokucdn.com/deploy/button.svg)](https://heroku.com/deploy)

This is a deployable server that allows you to use the Spotify Authorization Code Flow to create an application with the need to build your own server.

Key | Value
---  | ---
CLIENT_SECRET | Your client secret
CLIENT_ID | Your client id
APP_URL | URL for your front-end app
SERVER_URL | URL for the deployed herokuapp

When you deploy to Heroku go to your application in the dashboard and click on the settings tab. There you can click `Reveal Config Vars`, and make sure you add the four from above.

## Workflow

Here is the workflow for working with this server in your front-end application.

Create a link that goes to the deployed servers URL and the `/auth` endpoint.

```html
 <a href="https://someapp.herokuapp.com/auth">Login to Spotify</a>
```

Where `https://someapp.herokuapp.com` is your URL for the heroku server.

This will send the request to Spotify, which then redirects to the servers `/redirect` endpoint. From there it will take the returned data and then send the proper info to the next stop, which is the token step. Once that is completed it will redirect to your application with an object that looks like this as a query string.

On Spotify in the application dashboard for the API click the `edit settings` button and make sure you add `https://someapp.herokuapp.com/redirect` in the redirects URI's section.

```js
{
    access_token: "BQC5M6sBvrZBV3PFcICDKYyTnlfbbQ6...8WLXdEok",
    token_type: "Bearer",
    expires_in: 3600,
    refresh_token: "AQA1hxMB8FwWz3oqTegdK809jrYuVp-K...4AFgJoo"
}
```

However in order to get this, we will need to take the query string and grab what we need from it. In your JS on load you should have something like.

```js
const URL = new URL(location.href);
const tokens = {
    access_token: url.searchParams.get('access_token'),
    refrehs_token: url.searchParams.get('refresh_token')
};
```

```js
const getToken = () => {
    return new Promise((resolve,reject) => {
        $.ajax({
            url: 'http://localhost:3400/refresh',
            data: {
                refresh_token: tokens.refresh_token
            }
        })
        .then((res) => {
            const { access_token } = JSON.parse(res);
            resolve(access_token);
        }); 
    });
};

const getMe = () => {
    getToken()
        .then((token) => {
            $.ajax({
                url: 'https://api.spotify.com/v1/me',
                headers: {
                    'Authorization' : `Bearer ${token}`
                }
            })
            .then(console.log);
        });
};
```