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


