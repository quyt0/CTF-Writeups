# Javascript Puzzle

It is often useful to force exceptions to potentially get back valuable information.

Can you make a request which causes an exception in this app?

https://js-puzzle-974780027560.us-east5.run.app

@author: SamXML

---

The challenged provided a Javascript file named `app.js` which is running on ExpressJS

```JS
const express = require('express')

const app = express()
const port = 8000

app.get('/', (req, res) => {
    try {
        const username = req.query.username || 'Guest'
        const output = 'Hello ' + username
        res.send(output)
    }
    catch (error) {
        res.sendFile(__dirname + '/flag.txt')
    }
})

app.listen(port, () => {
    console.log(`Server is running at http://localhost:${port}`)
})
```

This code creates an `Express server` that listens on port `8000` and, when a `GET` request is made to `/`, it retrieves the `username` from the query string, sends back a greeting `Hello username` or sends the contents of `flag.txt` if an error occurs.

# Solution

Users can turn `username` into an object by sending a query like `?username[toString]`, which Express parses as

```JS
{ username: { toString: '' } }
```

And when we call `object + string`, JS will automatically call `object.toString() + string`

We can overwrite the toString method by make it a key of object. That is `object[toString]`

And once `object.toString()` is called, an error will occurs because `toString` is no longer a method but just a key

# Flag: `wctf{3xc3pt10n5_4r3_y0ur_fr13nd_14285137553}`

Send the payload `?username[toString]`, we got the flag