This past semester for my Database Design class two other students and I put together a culinary web app for tracking recipes and pantry inventory. To power the project, I built a Node backend using Express and mongoDB. The other members of my group used React to make a nice user interface, and I also threw together a simple Angular app to provide some admin features on the front end.

We ran into some issues early on putting the React and Node code in the same repo and having them play nicely together on Heroku, where we hosted the project. Once React development heated up, my two partners split their front end work into a second repository and hosted the product on a different server to reduce friction. After all, the point was to achieve an MVP that demonstrated our proficiency with databases, not to show off dev-ops skills.

It ended up being a good move for the scope of the project, and it allowed me to slide in an Angular app with minimal adjustments to the backend repo. The biggest drawback of splitting the repos up was how it affected backend API calls from the React app. First, we needed to enable Cross-Origin Resource Sharing in the backend app to allow requests from the React domain. Next, it led to hard-coding the API calls hostname to that of the Heroku server, which did not easily allow switching to a local backend for local development.

The latter could probably be dome with environment variables, but that isn't an ideal solution since the local machine would need to run two servers at once.

So in trying to make this situation better (as an exercise, after the project was graded!) I set out to combine these two repos and deploy them together on Heroku. But I also didn't just want to copy the React files into the backend repo, thereby losing valuable commit history.

## Combining Unrelated Git Repositories

This proved to be a little tricky since I was essentially merging two branches with absolutely no related history, but fortunately I found some instructions and figured out how to make it work for me. I started with the backend repo since I wanted to retain the Heroku deployment I had been using and went ahead merging some files in from the React app.

`cd ~/recipe-man # local backend working directory`

At this point, I jumped over to this directory in Finder (`open .`) and put every item inside a subdirectory called `backend`. This is to avoid any merge conflicts - there is no obvious way to actually combine the files into a working project (we'll get to this later), so I would rather have Git just get the files in the same repo and then get out of the way. We'll build the files back into a project manually later.

I forgot to transfer hidden files too - `.gitignore` gave me a merge conflict down the road.

Next I committed those changes - just moving every backend file from `~/recipe-man` to `~/recipe-man/backend`.

`git commit -am "Moved files to subdirectory"`

Now let's grab the React files.

`git remote add frontend <frontend-remote>`

`git merge --allow-unrelated-histories frontend/master`

The `--allow-unrelated-histories` option is very important - without it, Git throws an error. Next I moved all the new front end to a subdirectory of their own - `frontend`. Then I expanded the backend files back into the top level before checking in the changes so I could revert back if the next steps went wrong.

## React, Express, Angular, oh my!

The Express files were then in the `recipe-man` subdirectory, and the Angular app lives in `public/angular`. I added a new subdirectory `client` to the top level to place the React sources - the new `client/src` and `client/public` subdirectories, as well as some other package files. Those subdirectories and files come directly from my merged front end project.

The most important changes to make React play nicely with the backend and Angular apps were in `server.js` - the main script fired by Node:

First, the `client` subdirectory needed to be exposed publicly. When the server returns the base React HTML file, the browser needs to be able to load the accompanying files - like the JS files required to make the app run properly.

`app.use(express.static(__dirname + '/client/build'));` worked fine for me - I put it right below the similar statement for the `public` subdirectory, the one used to serve the Angular app. This way, the apps can live in separate subdirectories without interfering.

Next, let the server send down the React files when necessary. Some paths will trigger the Angular app and others will trigger the Express API endpoints - now it's time to let everything else send the base React file. As best as I understand, React runs in a single page like Angular, and the JS components manipulate the DOM to change what the page looks like. The Web address path can be a variety of things - it probably specifies to some degree what the React app is doing - but in all cases we want to return that single HTML file.

So I just need to add one more Express endpoint below all the others as a final catch all:
```javascript
app.get('*', function (req, res) {
    res.sendFile(__dirname + '/client/build/index.html');
});
```

At this point, I could also go through and cut out the hostname from each API call in the React app.

First, I added this property to `client/package.json`:

`"proxy": "http://localhost:5000"`

I think that just specifies this host as the backend to use.

Then, for example, `http://recipeman.jackfrysinger.com/api/users` can become `/api/users`.

## Build and Run
The React JS components need to be compiled into a minified JS script for execution. I wanted to make sure it worked fine locally before fiddling with my Heroku deployment.

I ran `npm install` in `~/recipe-man`, but that wasn't enough to get React going. In `~/recipe-man/client` I ran `npm install` again to get all the correct dependencies sorted, then `npm run build` created the JS build.

Then a quick trip back to `~/recipe-man` and firing off `npm start` did the job. After making sure `mongod` was running, of course.

## Shipping to Heroku
Turns out I just needed to add one property to the top-level `package.json`. Under scripts, I added

`"heroku-postbuild": "cd client && npm install --only=dev && npm install && npm run build"`

which just lets Heroku know what to run to get the repo ready for production. It makes sure the dependencies are installed and kicks off the React build. Once the React project is built into the client files, the Express endpoint just serves them statically.

I had auto Heroku deployments turned on, so the project was ready shortly after pushing!

## Deliverables
The [Git repo](https://github.com/jackfrys/recipe-man) and [Heroku deployment](http://recipeman.jackfrysinger.com).
