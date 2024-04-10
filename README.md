[<-- BACK](https://github.com/bkieselEducational/OAuth-SDKs)
# OAuth-2.0-Client-Library-for-Node.js

### Step 1: Register your app on the Google Cloud Platform and generate your credentials.
[Google Cloud Platform OAuth Setup](https://github.com/bkieselEducational/OAuth-Google-Cloud-Console-Setup)

### Step 2: Install the only requisite package:

```bash
$  npm install google-auth-library
```

### Step 3: You will need to alter your Model and Migration files for the User by making the password field NULLable:

*Migration File*
```javascript
module.exports = {
  async up(queryInterface, Sequelize) {
    await queryInterface.createTable("Users", {
      id: {
        allowNull: false,
        autoIncrement: true,
        primaryKey: true,
        type: Sequelize.INTEGER
      },
      ...
      ...
      hashedPassword: {
        type: Sequelize.STRING.BINARY,
        allowNull: true // THIS is what we need!!
      },
      ...
      ...
    }, options);
  },

  async down(queryInterface, Sequelize) {
    options.tableName = "Users";
    return queryInterface.dropTable(options);
  }
};
```

*Model File*
```javascript
module.exports = (sequelize, DataTypes) => {
  class User extends Model {
    static associate(models) {
      // define association here
    }
  };

  User.init(
    {
      ...
      ...
      hashedPassword: {
        type: DataTypes.STRING.BINARY,
        allowNull: true, // THIS is what we need!!
        validate: {
          len: [60, 60]
        }
      }
    },
    {
      sequelize,
      modelName: "User",
      defaultScope: {
        attributes: {
          exclude: ["hashedPassword", "email", "createdAt", "updatedAt"]
        }
      }
    }
  );
  return User;
};
```
### Step 4: Choose or create a new route file for the 2 necessary endpoints for an OAuth flow. (I am using session.js)
i. Firstly, we must handle any necessary imports within the route file and bring in any environment variables necessary for the flow.<br>
<br>
```javascript
const dotenv = require('dotenv');
dotenv.config();
```
```javascript
/*  Google AUTH  **********************************************/

const oauthClient = process.env.GOOGLE_OAUTH_CLIENT_ID
const oauthSecret = process.env.GOOGLE_OAUTH_CLIENT_SECRET

const {OAuth2Client} = require('google-auth-library');

// For saving some state for out OAuth flow (mostly PKCE related)!!
let oAuthState = {
  codeVerifier: '',
  nonce: '',
  state: ''
};

// A function for generating our random values for state and nonce.
function generateRandomString(length) {
  const charset = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
  let randomString = '';

  for (let i = 0; i < length; i++) {
    const randomIndex = Math.floor(Math.random() * charset.length);
    randomString += charset[randomIndex];
  }

  return randomString;
}

/**************************************************************/
```
ii. Next we must instantiate our first endpoint which will configure an OAuth client for us and begin the flow!
```javascript
router.get('/googleOauthLogin', async (req, res) => {
  // Your redirect URI will look different!
  const redirectUri = 'http://localhost:8000/api/sessions/googleOauthCallback';

  // Configure our Client class
  const oAuth2Client = new OAuth2Client(
    oauthClient,
    oauthSecret,
    redirectUri
  );

  // Generate the codeVerifier and codeChallenge for our PKCE security layer!
  const { codeVerifier, codeChallenge } = await oAuth2Client.generateCodeVerifierAsync();

  // Save our state for verification later.
  oAuthState.codeVerifier = codeVerifier;
  oAuthState.state = generateRandomString(16); // Generate a random state string
  oAuthState.nonce = generateRandomString(32); // Generate a random nonce string

  // Have our SDK generate the URL encoded with the appropriate values to start the flow.
  const authorizationUrl = oAuth2Client.generateAuthUrl({
    access_type: 'offline', // Forces a Refresh token to be sent!
    scope: 'https://www.googleapis.com/auth/userinfo.profile openid email',
    prompt: 'select_account consent', // select_account is necessary to garauntee that a user is prompted for account selection even when logged in to Google.
    state: oAuthState.state, // CSRF
    nonce: oAuthState.nonce, // CSRF
    code_challenge_method: 'S256', // PKCE
    code_challenge: codeChallenge  // PKCE
  });

  res.redirect(authorizationUrl); // Send a redirect to the client's browser which will fetch the Google Select an Account menu.
});
```

iii. Now we must setup the famous RedirectURI endpoint where we will receive our code from Google and trade it for an Access Token.
```javascript

router.get('/googleOauthCallback', async (req, res) => {
  // For the object that shall be returned from OpenID as a JWT payload!
  let claims = {};
  const code = req.query.code; // The ephemeral code returned by Google.

  // Our CSRF for the flow
  const googleState = req.query.state;
  if (googleState !== oAuthState.state) {
    res.status(500).json({ error: 'Could not verify client' });
  }

  try {
    const redirectUrl = 'http://localhost:8000/api/sessions/googleOauthCallback';

    const oAuth2Client = new OAuth2Client(
      oauthClient,
      oauthSecret,
      redirectUrl
    );

    const params = {
      code: code,
      codeVerifier: oAuthState.codeVerifier, // Pass the code_verifier
    };

    const res = await oAuth2Client.getToken(params); // Exchange code for access token.
    await oAuth2Client.setCredentials(res.tokens); // Set the credentials property of our Client class

    const user = oAuth2Client.credentials;

    const idToken = user.id_token; // The token from OpenID Connect
    const expiry = user.expiry_date;

    // Verify that the JWT signature is valid!
    const ticket = await oAuth2Client.verifyIdToken({idToken, oauthClient, expiry});
    const payload = ticket.getPayload();

    // More CSRF! Verify our nonce from earlier.
    const openIdNonce = payload.nonce;
    if (openIdNonce !== oAuthState.nonce) {
      res.status(500).json({ error: 'Could not verify client!' });
    }

    claims = payload;
  } catch(err) {
    console.log('Could not sign in with Google', err);
  }

  const gmail = claims.email; // Extract the user's email from the ID Token claims!

  // Check to see if a user with this email address already exists
  let user = await User.unscoped().findOne({
    where: {
      [Op.or]: {
        username: gmail,
        email: gmail
      }
    }
  });

  let safeUser = {};
  if (user) {
    safeUser = {
      id: user.id,
      email: user.email,
      username: user.username,
    };
  } else {
    // If no user exists, we shall create a new one!
    user = await User.create({ email: gmail, username: claims.name }); // Note: No password!!!

    safeUser = {
      id: user.id,
      email: user.email,
      username: user.username,
    };
  }

  // Let's garbage collect those random values!!
  oAuthState = {
    codeVerifier: '',
    nonce: '',
    state: ''
  };

  await setTokenCookie(res, safeUser);

  return res.redirect('http://localhost:5173/'); // Back to the browser!
});
```

### Step 5: And finally, we must create a button in our frontend code that the user can click to initiate the flow!
```javascript
  <a href={`${window.origin}/api/session/googleOauthLogin`}><button>OAUTH</button></a>
```
