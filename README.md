# Pepito
Sh on top
from flask import Flask, redirect, session, url_for, request
import google_auth_oauthlib.flow
import json
import os
import requests

app = Flask('app')
# `FLASK_SECRET_KEY` is used by sessions. You should create a random string
# and store it as secret.
app.secret_key = os.environ.get('FLASK_SECRET_KEY') or os.urandom(24)

# `GOOGLE_APIS_OAUTH_SECRET` contains the contents of a JSON file to be downloaded
# from the Google Cloud Credentials panel. See next section.
oauth_config = json.loads(os.environ['GOOGLE_OAUTH_SECRETS'])

# This sets up a configuration for the OAuth flow
oauth_flow = google_auth_oauthlib.flow.Flow.from_client_config(
    oauth_config,
    # scopes define what APIs you want to access on behave of the user once authenticated
    scopes=[
        "https://www.googleapis.com/auth/userinfo.email",
        "openid", 
        "https://www.googleapis.com/auth/userinfo.profile",
    ]
)

# This is entrypoint of the login page. It will redirect to the Google login service located at the
# `authorization_url`. The `redirect_uri` is actually the URI which the Google login service will use to
# redirect back to your app after the user logs in.
@app.route('/login')
def login():
    authorization_url, state = oauth_flow.authorization_url(
        # Enable offline access so that you can refresh an access token without
        # re-prompting the user for permission.
        access_type='offline',
        # Enable ID token so you can verify the user's identity
        include_granted_scopes='true'
    )
    session['state'] = state
    return redirect(authorization_url)

# This is the callback URL, which is specified in the Google Cloud Console. It will
# be called by Google's OAuth service after the user has successfully logged in.
@app.route('/oauth2callback')
def callback():
    # If there is no state in the session, this means the user did not start
    # the authorization process.
    if 'state' not in session:
        return 'Error: State not found in session', 400

    state = session['state']
    # The `request.url` is used to get the current URL, which contains the code
    # that Google provides after the user has logged in.
    oauth_flow.fetch_token(authorization_response=request.url, state=state)
    # This is the access token that Google provides. 
    credentials = oauth_flow.credentials
    session['google_credentials'] = credentials_to_dict(credentials)
    return redirect('/')

# This function is used to convert the credentials object to a dictionary,
# which is then stored in the session. This is done because the credentials object
# cannot be stored in the session directly.
def credentials_to_dict(credentials):
    return {'token': credentials.token,
            'refresh_token': credentials.refresh_token,
            'token_uri': credentials.token_uri,
            'client_id': credentials.client_id,
            'client_secret': credentials.client_secret,
            'scopes': credentials.scopes}

# This route is used to display the user's information after they have logged in.
# It retrieves the user's information from Google using the access token.
@app.route('/')
def index():
    if 'google_credentials' not in session:
        return redirect('/login')
    credentials = google_auth_oauthlib.flow.Credentials(**session['google_credentials'])
    # This is the user's information from Google's API.
    userinfo = google_auth_oauthlib.flow.InstalledAppFlow.from_client_config(
        oauth_config,
        scopes=["https://www.googleapis.com/auth/userinfo.profile", "https://www.googleapis.com/auth/userinfo.email"]
    ).credentials.refresh(credentials)

    return f"Welcome, {userinfo['given_name']}! Your email is {userinfo['email']}."

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)