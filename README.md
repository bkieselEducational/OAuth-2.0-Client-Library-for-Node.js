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

// For the object that shall be returned from OpenID as a JWT payload!
let claims = {};

/**************************************************************/
```
ii.
