---
title: "How to build an Authentication system using JWT in NodeJS: Part 2"
date: "2022-06-13T23:46:37.121Z"
template: "post"
draft: false
slug: "/posts/how-to-build-an-authentication-system-using-JWT-in-nodeJS:-part-2"
category: "Software Engineering"
tags:
  - "Web Development"
description: "Authentication is an important part when building applications. In this article series, I'll show you how to build an efficient authentication system."
---

In the [**Part 1**](https://hashnode.com/draft/629768228ec91103236a3597) of this article series, we introduced **JWT** (JSON Web Token) and went further to implement an API endpoint that sends a token when a new user signs up on our application.

At the end of this series, we'll have built these APIs listed below:

| METHOD      | URL                               | ACTION                         |
| :---                |    :---                          |          ---:                        |
| POST ✅        | /auth/signup           | Signup a new acount   |
| POST            | /auth/verification  | Verify a new account   |
| POST            | /auth/login                       | Login verified account |
| GET               | /user/info        | Get user private data   |
| PUT               | /auth/forgot-password   | Forgot Password          |
| PUT               | /auth/reset-password/:token | Verify password reset token |

Previously, we implemented an endpoint to signup a new account on our application. Now let's verify the token that was sent by the API.

### Step 3: Implementation of authentication APIs. (Cont'd)

- **Step 3.4:** In `src/controllers/auth.controller.ts` add `emailVerificationController` endpoint and update the export.

```
/**
 * API endpoint to verify user signup token.
 * @returns response with a success of true if token is valid else false if invalid.
*/
const emailVerificationController = async (req: Request, res: Response) => {
  const { error } = tokenRequestValidator.validate(req.body);
  if (error) {
    return res.status(400).json({
      success: false,
      message: error.details[0].message
    });
  }

  const { token } = req.body;
  if (token) {
    try {
      const decodedToken = jwt.verify(token, `${process.env.TOKEN_KEY}`) as jwt.JwtPayload;
      const { email, password } = decodedToken;

      // Check if this user has gone through this process and is already in DB
      const existingUser = await User.findOne({ email: email });
      if (existingUser) {
        return res.status(200).json({ 
          success: false,
          message: `${email} has already been verified.`
        });
      }

      // Hash user password
      const hashedPassword = await bcrypt.hash(password, 10);

      // Create new user and save to MongoDB
      const user = User.build({ 
        email: `${email}`, 
        password: `${hashedPassword}`
      });
      user.save();
      res.status(201).json({ success: true, id: user._id, message: 'User Registration Successful.' });
    } catch (error) {
      return res.status(400).json({ message: `${error}` });
    }
  } else {
    return res.status(408).json({ success: false, message: "No verification token attached." });
  }
};

export { 
  userSignupController,
  emailVerificationController
};
``` 

- **Step 3.5 :** Update `src/routes/auth.route.ts`

```
import express from 'express';
const router = express.Router();

import { 
  userSignupController, 
  emailVerificationController
} from '../controllers/auth.controller';

router.post('/signup', userSignupController);
router.post('/verification', emailVerificationController);

module.exports = router;
```

Using Postman to test our endpoint, as shown below.

![Screenshot 2022-06-06 at 16.51.28.png](/media/postman-verify.png)

 Alright now, let's implement the login API.

- **3.6 :** In `src/controllers/auth.controller.ts` 

```
/**
 * API endpoint to login a user.
 * @returns response with a success of true if login credentials are valid else false.
*/
const loginController = async (req: Request, res: Response) => {
  const { error } = userCredentialRequestValidator.validate(req.body);
  if (error) {
    return res.status(400).json({
      success: false,
      message: error.details[0].message
    });
  }

  const { email, password } = req.body;

  // Check for valid user email.
  const user = await User.findOne({ email });
  if (!user) {
    return res.status(401).json({
      success: false,
      message: "Email does not exists."
    });
  }

  // Confirm found user password.
  const isPasswordValid = await bcrypt.compare(password, user.password);
  if (!isPasswordValid) {
    return res.status(401).json({
      success: false,
      message: "Invalid password."
    });
  }
  
  // Generate a token and send to client.
  const token = jwt.sign({ _id: user._id }, `${process.env.TOKEN_KEY}`, { expiresIn: '30m' });
  res.status(200).json({ 
    success: true, 
    message: 'Login Successful',
    token, 
    user: user._id 
  });
};

export { 
  userSignupController,
  emailVerificationController,
  loginController
};
``` 

- **Step 3.7 :** Update `src/routes/auth.route.ts`

```
import express from 'express';
const router = express.Router();

import { 
  userSignupController, 
  emailVerificationController,
  loginController
} from '../controllers/auth.controller';

router.post('/signup', userSignupController);
router.post('/verification', emailVerificationController);
router.post('/login', loginController);

module.exports = router;
```

Now let's call the loginController on Postman as shown below.

![Screenshot 2022-06-06 at 17.02.38.png](/media/postman-login.png)

To help understand the next steps, we need to introduce the concept of **middleware**

**Middleware** are components that have access to the request object (req), the response object (res), and the next middleware function in the application’s request-response cycle. Check [here](https://selvaganesh93.medium.com/how-node-js-middleware-works-d8e02a936113) for further reading about middlewares.

![Screenshot 2022-06-01 at 16.00.19.png](/media/jwt-explain.png)

Now let's create a middleware function that would verify the token when a user makes a request to a private API on our application.

- **3.8 :** In `src/middlewares/verify.token.ts` 

```
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';

const verifyAuthToken = (req: Request, res: Response, next: NextFunction) => {
  const authHeader = req.headers.authorization;
  
  // Verify request has token attached.
  if (!authHeader) {
    return res.status(401).json({
      success: false,
      message: 'No token provided. Access denied!'
    });
  }

  try {
    const token = authHeader.split(' ')[1];
    jwt.verify(token, `${process.env.TOKEN_KEY}`);
    next();
  } catch (error) {
    return res.status(401).json({
      success: false,
      message: `${error}`
    });
  }
};

export { verifyAuthToken };
``` 

Now let's create a user API to retrieve private content from our application.

- **3.9 :** In `src/controllers/user.controller.ts` 

```
import { Request, Response } from 'express';

/**
 * API endpoint to get protected data from the application.
 * @returns response with success of true if the user has been authorized
*/
const userInfoController = async (req: Request, res: Response) => {
  res.status(200).json({ 
    success: true, 
    message: `Hello 👋, you've been authorized to access this endpoint.`  
  });
};

export { userInfoController };
```

- **3.10 :** In `src/routes/user.route.ts`  we've included the middleware function `verifyAuthToken` to verify the user's authorization token.

```
import express from 'express';
const router = express.Router();
import { verifyAuthToken } from '../middlewares/verify.token';

import { 
  userInfoController
} from '../controllers/user.controller';

router.get('/info', verifyAuthToken, userInfoController);

module.exports = router;
``` 

Update our `src/index.ts` file with the `userRouter` as shown below

```
import express from 'express';
import 'dotenv/config';
import mongoose from 'mongoose';

const app = express();
const port = process.env.PORT || 8080;
app.use(express.json());

// Routers
const authRouter = require('./routes/auth.route');
const userRouter = require('./routes/user.route');

app.use('/auth', authRouter);
app.use('/user', userRouter);  // < -- router update here

// Start server 
const startServer = async () => {
  try {
    await mongoose.connect(`${process.env.MONGO_URI}`);
    app.listen(port, () => {
      console.log(`Server listening on port: ${port}`);
    })
  } catch (err) {
    process.exit(1);
  }
};

startServer();
``` 

Now let's call the API endpoint to fetch private data, as shown below.

![Screen Recording 2022-06-06 at 17.gif](/media/token-explain.gif)

Alright, now we've seen how to create an API and make sure that only authorized users are allowed to access the resource. Now let's implement the forgot password API.

- **3.11 :** Update `src/controllers/auth.controller.ts` 

```
/**
 * API endpoint to send a forgot-password token to the user.
 * @returns response with a success of true if the token is sent to user's email else false.
*/
const forgotPasswordController = async (req: Request, res: Response) => {
  const { error } = forgotPasswordRequestValidator.validate(req.body);
  if (error) {
    return res.status(400).json({
      success: false,
      message: error.details[0].message
    });
  }

  const { email } = req.body;

  // Check if user exists
  const user = await User.findOne({ email });
  if (!user) {
    return res.status(401).json({
      success: false,
      message: "Email does not exists."
    });
  }
  
  // Generate Password Reset Token
  const token = jwt.sign({ _id: user._id }, `${process.env.TOKEN_KEY}`, { expiresIn: '5m' });

  const mailOptions = {
    from: `${process.env.EMAIL_FROM}`, 
    to: `${email}`,
    subject: 'Forgot password link.',
    html: `
      <p>Please copy and paste this link to postman. <b>( Reset Password )</b></p>
      <a>${token}</a>
    `
  };

  // Send mail with defined transport object
  mailTransporter.sendMail(mailOptions, (error: any) => {
    if (error) {
      return res.status(500).json({ success: false, message: `Something went wrong. Pls try again!` });
    }
    res.status(200).json({ 
      success: true, 
      message: `Email has been sent to ${email}. Follow the instruction to set a new password.`  
    });
  });
};

export { 
  userSignupController,
  emailVerificationController,
  loginController,
  forgotPasswordController,
};
``` 

- **Step 3.12 :** Update `src/routes/auth.route.ts`

```
import express from 'express';
const router = express.Router();

import { 
  userSignupController, 
  emailVerificationController,
  loginController,
  forgotPasswordController
} from '../controllers/auth.controller';

router.post('/signup', userSignupController);
router.post('/verification', emailVerificationController);
router.post('/login', loginController);
router.put('/forgot-password', forgotPasswordController);

module.exports = router;
```

Let's call the forget password endpoint.

![Screenshot 2022-06-06 at 17.46.26.png](/media/postman-password.png)

- **3.13 :** Now verify the forgotten password token sent, and update the user password.

`src/controllers/auth.controller.ts` 

```
/**
 * API endpoint to verify forgot-password token sent to user and reset password.
 * @returns response with a success of true if token is valid else false.
*/
const resetPasswordController  = async (req: Request, res: Response) => {
  const { token } = req.params;
  const { password1, password2 } = req.body;

  const { error } = passwordResetRequestValidator.validate(req.body);
  if (error) {
    return res.status(400).json({
      success: false,
      message: error.details[0].message
    });
  }

  try {
    const payload = jwt.verify(token, `${process.env.TOKEN_KEY}`) as jwt.JwtPayload;

    // Validate password match
    if (password1 !== password2) {
      return res.status(400).json({
        success: false,
        message: `Password does not match!` 
      });
    }

    // Find user with id and update with new password
    const user = await User.findById(payload._id);
    if (!user) {
      return res.status(401).json({
        success: false,
        message: "User ID not valid."
      });
    }

    // Hash password before update user document in DB
    const hashedPassword = await bcrypt.hash(password1, 10);
    user.password = hashedPassword;
    user.save();

    res.status(200).json({ 
      success: true,
      id: user._id,
      message: `Password update successful!` 
    });
  } catch (error) {
    return res.status(400).json({ success: false, message: `${error}` });
  }
}

export { 
  userSignupController,
  emailVerificationController,
  loginController,
  forgotPasswordController,
  resetPasswordController
};
``` 

- **Step 3.14 :** Let's update `src/routes/auth.route.ts`

```
import express from 'express';
const router = express.Router();

import { 
  userSignupController, 
  emailVerificationController,
  loginController,
  forgotPasswordController,
  resetPasswordController
} from '../controllers/auth.controller';

router.post('/signup', userSignupController);
router.post('/verification', emailVerificationController);
router.post('/login', loginController);
router.put('/forgot-password', forgotPasswordController);
router.put('/reset-password/:token', resetPasswordController);

module.exports = router;
```

Now let's reset our password by entering a new password, and pasting the token like `http://localhost:8080/auth/api/forgot-password/our-password-token` as shown below.

![Screenshot 2022-06-06 at 17.56.18.png](/media/postman-reset.png)


### Conclusion

In this article, you were introduced to JWTs and how it's used to build an efficient authentication system for your applications.

For more background on JWTs, there is the [Introduction to JSON Web Tokens](https://jwt.io/introduction/) documentation.

The source code for this project is also available on [Github](https://github.com/kapbyte/jwt-auth-tutorial)

Happy coding! 👨🏽‍💻👍🏽