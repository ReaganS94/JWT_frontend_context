# JWT + useContext

Starting repos:

- Backend: https://github.com/ReaganS94/JWT_Auth
- Frontend: https://github.com/ReaganS94/JWT_Frontend

Time to refactor the code to use React's more friendly Context API.

- In the `src` folder, create another folder called `context`, and in this newly created folder, a file named `authContext.js`

- Start by importing all the necessary things:

```js
import { useState, useEffect, createContext } from "react";
```

And create the `AuthContext` and `AuthContextProvider`

```js
export const AuthContext = createContext();

export default function AuthContextProvider(props) {
  return (
    <AuthContext.Provider value={{}}>{props.children}</AuthContext.Provider>
  );
}
```

- Time to build up the context! Create a state named `token` and initialize it as `null`

- Now we'll need two useEffects!

  - The **first** `useEffect` is responsible for initializing the token state when the component is first mounted. It retrieves the token value from the local storage using `localStorage.getItem("token")` and updates the token state with the stored token value if it exists.

```js
useEffect(() => {
  const storedToken = localStorage.getItem("token");
  console.log("storedToken", storedToken);
  if (storedToken) {
    setToken(storedToken);
  }
}, []);
```

- The **second** `useEffect` monitors changes to the token state and performs actions based on those changes. When the token value changes, it updates the corresponding value in the local storage using `localStorage.setItem("token", token)`. If the token is `null`, it removes the corresponding value from the local storage using `localStorage.removeItem("token")`. By including `[token]` as the dependency array, this effect will run whenever the token value changes.

```js
useEffect(() => {
  if (token) {
    localStorage.setItem("token", token);
  } else {
    localStorage.removeItem("token");
  }
}, [token]);
```

- Let's also create two functions, one to login and one to logout. The `login` function should take a token as a parameter and use the `setToken` to change the `token` state to the argument we pass to it.
  The `logout` simply needs to set the `token` state to `null`

```js
const login = (newToken) => {
  setToken(newToken);
};

const logout = () => {
  setToken(null);
};
```

- Remember to pass everything we need to make available within this context to the `value` attribute. We're going to need `token`, `login` and `logout`.

```js
return (
  <AuthContext.Provider value={{ token, login, logout }}>
    {props.children}
  </AuthContext.Provider>
);
```

- Here's how the context should look like at the end:

```js
import { useState, useEffect, createContext } from "react";

export const AuthContext = createContext();

export default function AuthContextProvider(props) {
  const [token, setToken] = useState(null);

  useEffect(() => {
    const storedToken = localStorage.getItem("token");
    console.log("storedToken", storedToken);
    if (storedToken) {
      setToken(storedToken);
    }
  }, []);

  useEffect(() => {
    if (token) {
      localStorage.setItem("token", token);
    } else {
      localStorage.removeItem("token");
    }
  }, [token]);

  const login = (newToken) => {
    setToken(newToken);
  };

  const logout = () => {
    setToken(null);
  };

  return (
    <AuthContext.Provider value={{ token, login, logout }}>
      {props.children}
    </AuthContext.Provider>
  );
}
```

- Now, just like any other context, in order to be able to use it we need to wrap our `App.js` (or any component/s we want to give access to the contents of this context) in it. So, in `index.js`:

```js
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App";
import { BrowserRouter } from "react-router-dom";
import AuthContextProvider from "./context/authContext";

const root = ReactDOM.createRoot(document.getElementById("root"));
root.render(
  <React.StrictMode>
    <BrowserRouter>
      <AuthContextProvider>
        <App />
      </AuthContextProvider>
    </BrowserRouter>
  </React.StrictMode>
);
```

- Time to put the context we just created to use. Let's start from `Home.js`. Open the file and make the following changes:
  - Import `AuthContext` and `useContext`
  - Destructure `token` from `AuthContext`
  - Swap all instances of `user` (the prop that was being passed to this component) to `token`
  - While we're at it, let's also create a loading message while we wait for the posts

```js
import { useState, useEffect, useContext } from "react";
import { AuthContext } from "../context/authContext";

export default function Home() {
  const [posts, setPosts] = useState([]);
  const [loading, setLoading] = useState(true);

  const { token } = useContext(AuthContext);

  useEffect(() => {
    const getData = async () => {
      try {
        const res = await fetch("http://localhost:8080/posts", {
          headers: {
            Authorization: `Bearer ${token}`,
          },
        });
        const data = await res.json();
        setPosts(data);
        setLoading(false);
      } catch (error) {
        console.log(error);
        setLoading(false);
      }
    };
    if (token) {
      getData();
    }
  }, [token]);

  return (
    <div className="posts">
      {loading ? ( // Show loading message if loading is true
        <h1>Loading...</h1>
      ) : (
        <>
          {posts.length ? (
            posts.map((post) => (
              <div
                key={post._id}
                style={{ border: "2px solid black", margin: "10px" }}
              >
                <h2>{post.title}</h2>
                <p>{post.body}</p>
              </div>
            ))
          ) : (
            <h1 style={{ color: "red" }}>No posts found</h1>
          )}
        </>
      )}
    </div>
  );
}
```

- On to the next file, `Login.js`

  - Let's start by importing `AuthContext` and `useContext`
  - Destructure `login` from the `AuthContext`
  - In the `handleSubmit` function, more specifically in the second if statement, the `if (response.ok)`, instead of stringifying the whole data and setting it into storage, we'll only set the token. Let's also change the storage key from `user`to `token`, as this will be more accurate since we're only storing the token
  - After that, let's call the `login` function we grabbed from context and pass it the token
  - One more thing. The loading time is super quick right now, so there's almost no waiting time from the moment we click on "Login" to the page redirect as both the backend and frontend are running on localhost, but when working with deployed projects, things will be significantly slower. I have seen far too many project presentations where someone clicks on "Login", nothing happens for a few seconds, panic start to settle in, you harass the "Login" button while nervously chuckling and saying "this was working just half an hour ago, I swear". To avoid all this we'll make use of that `isLoading` and a cool package called "React-Loading-Overlay" (https://www.npmjs.com/package/react-loading-overlay#quick-start-running_woman). Unfortunately it hasn't been updated in a long time, so we can't just install it regularly. We'll have to basically tell React that we're aware that this package is old but we want it anyway. To do that, run this command in your terminal:
    `npm i --legacy-peer-deps react-loading-overlay`
    This should allow it to work just fine. Here's how your `Login.js` should look like when it's all said and done (I have also added a 5 seconds timeout for the login so that we get to see the loading spinner in action, but do remember to remove it if you copy this code for your own project):

```js
import { useState, useContext } from "react";
import { AuthContext } from "../context/authContext";
import LoadingOverlay from "react-loading-overlay";

export default function Login() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState(null);
  const [isLoading, setIsLoading] = useState(false);

  const { login } = useContext(AuthContext);

  const handleSubmit = async (e) => {
    e.preventDefault();

    setIsLoading(true);
    setError(null);

    const response = await fetch("http://localhost:8080/user/login", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ email, password }),
    });

    const data = await response.json();

    if (!response.ok) {
      setIsLoading(false);
      setError(data.error);
    }

    if (response.ok) {
      setTimeout(() => {
        localStorage.setItem("token", data.token);
        setIsLoading(false);
        login(data.token);
      }, 5000);
    }
  };

  return (
    <LoadingOverlay active={isLoading} spinner text="Logging in...">
      <form className="login" onSubmit={handleSubmit}>
        <h3>Log in</h3>
        <label>email: </label>
        <input
          type="email"
          onChange={(e) => setEmail(e.target.value)}
          value={email}
        />

        <label>password: </label>
        <input
          type="password"
          onChange={(e) => setPassword(e.target.value)}
          value={password}
        />

        <button>Log in</button>
        {error && <div className="error">{error}</div>}
      </form>
    </LoadingOverlay>
  );
}
```

- Let's move on to `Signup.js`. This will be an almost 1:1 copy on `Login.js`, so I'll just give you the code without any further boring long text. I guess the reason for that is because I believe in you by now, we're almost at the end of the bootcamp so I can leave a few things here and there without fully explaining every single step. What I'm trying to say is that I believe in you! One more thing, you'll notice there's also a username in there now, no worries, be patient, it's all part of the plan. Ok ok, here's the code:

```js
import { useState, useContext } from "react";
import { AuthContext } from "../context/authContext";
import LoadingOverlay from "react-loading-overlay";

export default function Signup({ setUser }) {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [username, setUsername] = useState("");
  const [error, setError] = useState(null);
  const [isLoading, setIsLoading] = useState(false);

  const { login } = useContext(AuthContext);

  const handleSubmit = async (e) => {
    e.preventDefault();

    setIsLoading(true);
    setError(null);

    const response = await fetch("http://localhost:8080/user/signup", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ email, password, username }),
    });

    const data = await response.json();

    if (!response.ok) {
      setIsLoading(false);
      setError(data.error);
    }

    if (response.ok) {
      localStorage.setItem("token", data.token);
      setIsLoading(false);
      login(data.token);
    }
  };

  return (
    <LoadingOverlay active={isLoading} spinner text="Signing in...">
      <form className="signup" onSubmit={handleSubmit}>
        <h3>Sign up</h3>
        <label>username: </label>
        <input
          type="text"
          onChange={(e) => setUsername(e.target.value)}
          value={username}
        />

        <label>email: </label>
        <input
          type="email"
          onChange={(e) => setEmail(e.target.value)}
          value={email}
        />

        <label>password: </label>
        <input
          type="password"
          onChange={(e) => setPassword(e.target.value)}
          value={password}
        />

        <button>Sign up</button>
        {error && <div className="error">{error}</div>}
      </form>
    </LoadingOverlay>
  );
}
```

- Let's move on to `Navbar.js`. Here's what's to do:

  - Import the usual `AuthContext` and `useContext`

  - Destructure `logout` and `token` from the context
  - In the `handleClick` function, change the argument of `removeItem` from "user" to "token"
  - Get rid of `setUser` and call the `logout` function instead
  - Install a super cool package called `react-jwt` (npm i react-jwt)
  - If this install also gives you trouble, you can either use the `--legacy-peer-deps` flag OR uninstall react-loading-overlay, install react-jwt normally, and then re-install react-loading-overlay with the `--legacy-peer-deps` again
  - Import `useJwt`
  - Destructure `decodedToken` from `useJwt(token)`
  - change all instances of user to `token`
  - At the top of the navbar we were displaying the logged in user's email, this won't be possible anymore as we're now reading the payload from the token itself. Does that mean that we're gonna do without? Heck no, we're gonna make the necessary changes to make it so that users have to also create a username when they signup and we'll display that username here at the top. First I'll give you the code for `Navbar.js` but keep in mind that it won't work without some changes to the backend!

```js
import { useContext } from "react";
import { AuthContext } from "../context/authContext";
import { Link } from "react-router-dom";
import { useJwt } from "react-jwt";

export default function Navbar() {
  const { logout, token } = useContext(AuthContext);

  const handleClick = () => {
    localStorage.removeItem("token");
    logout();
  };

  const { decodedToken } = useJwt(token);

  return (
    <div className="container">
      <div className="title">
        <Link to="/">My Cool Blog</Link>
      </div>
      <nav>
        {token !== null && (
          <div>
            <span style={{ padding: "10px" }}>Hello, {decodedToken?.name}</span>
            <button onClick={handleClick}>Log out</button>
          </div>
        )}
        {token === null && (
          <div>
            <Link to="login">Login</Link>
            <Link to="signup">Signup</Link>
          </div>
        )}
      </nav>
    </div>
  );
}
```

- Buckle up, it's time to get our hand back into the backend! To make things work properly, we not only need to create a new field for the username, but we'll have to also add it to the things (more like one thing in our case, the \_id) that we sign when creating the JWT. Here goes!
- Let's start with the User's **schema**. Add another field named "username" to the existing schema.
- The login static method can stay untouched, but the signup needs to change in order to accommodate the username

```js
const userSchema = new mongoose.Schema({
  email: {
    type: String,
    required: true,
    unique: true,
  },
  password: {
    type: String,
    required: true,
  },
  username: {
    type: String,
    required: true,
  },
});

// creating a custom static method

userSchema.statics.signup = async function (email, password, username) {
  //validation

  const exists = await this.findOne({ email });

  if (exists) {
    throw Error("Email already in use");
  }

  if (!email || !password || !username) {
    throw Error("All fields must be filled");
  }

  if (!validator.isEmail(email)) {
    throw Error("email is not valid");
  }

  if (!validator.isStrongPassword(password)) {
    throw Error(
      "Make sure to use at least 8 characters, one upper case letter, a number and a symbol"
    );
  }

  const salt = await bcrypt.genSalt(10);

  const hash = await bcrypt.hash(password, salt);

  const user = await this.create({ username, email, password: hash });

  return user;
};
```

- On to the `userControllers`. Here we have a few things to change. First thing is the function that we use to create the token. As I mentioned before, we don't want to only sign the `id` anymore, we also want to sign the username so that it becomes part of the token's payload and we can access it in the frontend.

```js
const createToken = (_id, name) => {
  return jwt.sign({ _id, name }, process.env.SECRET, { expiresIn: "1d" });
};
```

- We need to change the call to this function in both the login and signup functions so that it signs the username as well

```js
// login user
const loginUser = async (req, res) => {
  const { email, password } = req.body;

  try {
    const user = await User.login(email, password);

    //create token
    const token = createToken(user._id, user.username);

    res.status(200).json({ email, token });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

// sign up user
const signUpUser = async (req, res) => {
  const { email, password, username } = req.body;

  try {
    const user = await User.signup(email, password, username);
    console.log(user);
    //create token
    const token = createToken(user._id, user.username);
    res.status(200).json({ email, token, username });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};
```

- Great, that's all the changes we needed to make in the backend. Now we should have the username in the token. Keep in mind this won't work with any of the users that you have created so far, so please go ahead and nuke your `users` collection and create a new user using the Signup.

- We're almost done! Now let's go back to `App.js` and add the final touches.

- You guessed it, import `AuthContext` and `useContext`

- Delete everything before the return statement

- Destructure `token` from the context

- Swap all instances of "user" to "token"

- While we're at it, let's also create a "catch all" route

```js
import { useContext } from "react";
import { Routes, Route, Navigate } from "react-router-dom";
import Home from "./components/Home";
import Navbar from "./components/Navbar";
import Login from "./components/Login";
import Signup from "./components/Signup";
import { AuthContext } from "./context/authContext";
import "./App.css";

function App() {
  const { token } = useContext(AuthContext);

  return (
    <div className="App">
      <Navbar />
      <Routes>
        <Route path="/" element={token ? <Home /> : <Navigate to="/login" />} />
        <Route
          path="/login"
          element={!token ? <Login /> : <Navigate to="/" />}
        />
        <Route
          path="/signup"
          element={!token ? <Signup /> : <Navigate to="/" />}
        />
        <Route path="*" element={<Navigate to="/" />} />
      </Routes>
    </div>
  );
}

export default App;
```

Aaaand we're done! Good job, I mean it. Go have a look at what you just created, play around with it for a bit but then make sure to take a break! Remember that you're not supposed to remember all this by heart or be able to reproduce this whole thing without looking things up. Use this guide whenever you want to build a frontend that leverages JWT!

### EXTRA GOAL

One more thing before you go. When you're done with your break, implement a way to create posts from within the app. The controller for creating a post already exists, all you have to do is use it!
