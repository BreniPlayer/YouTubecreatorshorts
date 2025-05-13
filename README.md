pip install Flask google-auth-oauthlib google-auth requests
import os
import pathlib
import google_auth_oauthlib.flow
import googleapiclient.discovery
import googleapiclient.errors
from flask import Flask, session, redirect, url_for, request

app = Flask(__name__)
app.secret_key = "chave_segura"

CLIENT_SECRETS_FILE = "client_secret.json"
SCOPES = ["https://www.googleapis.com/auth/youtube.readonly"]

@app.route("/")
def index():
    if "credentials" not in session:
        return '<a href="/login">Conectar com o YouTube</a>'
    return redirect("/canal")

@app.route("/login")
def login():
    flow = google_auth_oauthlib.flow.Flow.from_client_secrets_file(
        CLIENT_SECRETS_FILE, scopes=SCOPES)
    flow.redirect_uri = url_for('oauth2callback', _external=True)

    authorization_url, state = flow.authorization_url(
        access_type='offline', include_granted_scopes='true')

    session['state'] = state
    return redirect(authorization_url)

@app.route("/oauth2callback")
def oauth2callback():
    state = session['state']

    flow = google_auth_oauthlib.flow.Flow.from_client_secrets_file(
        CLIENT_SECRETS_FILE, scopes=SCOPES, state=state)
    flow.redirect_uri = url_for('oauth2callback', _external=True)

    flow.fetch_token(authorization_response=request.url)

    credentials = flow.credentials
    session['credentials'] = credentials_to_dict(credentials)

    return redirect("/canal")

@app.route("/canal")
def canal():
    if 'credentials' not in session:
        return redirect('/')

    credentials = google.oauth2.credentials.Credentials(**session['credentials'])
    youtube = googleapiclient.discovery.build('youtube', 'v3', credentials=credentials)

    request_api = youtube.channels().list(part='snippet,statistics', mine=True)
    response = request_api.execute()

    canal = response['items'][0]['snippet']['title']
    return f'Você está logado como: {canal}'

def credentials_to_dict(credentials):
    return {
        'token': credentials.token,
        'refresh_token': credentials.refresh_token,
        'token_uri': credentials.token_uri,
        'client_id': credentials.client_id,
        'client_secret': credentials.client_secret,
        'scopes': credentials.scopes
    }

if __name__ == "__main__":
    app.run("localhost", 5000, debug=True)
