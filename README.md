routines
========

A simple yet powerful 'start' function for initiating coroutines written using the es6 yield keyword.

This is meant to work in javascript 1.7 or node > v0.11.

The psuedo-recursive magic start function
========================================
```coffeescript
start = (routine, callback = ((error, data)-> throw error if error), error, data)->
  try
    # either resume the routine with an exception if passed an error (routine.throw error),
    # or resume it normaly with data (routine.next data)
    # try/catch blocks wrapping the yield statements of routines will first catch this error,
    # otherwise if none are implemented, it will bubble back up this try/catch
    {done, value} = if error then routine.throw error else routine.next data
    unless done then value (error, data...)->
    
      # pseudo-recursive start next step of routine
      start routine, callback, error, switch data.length
        when 0 then undefined # switch data into undefined since no data was provided
        when 1 then data[0] # switch data into a single value since only one value was provided
        else data # since iterators may not be resumed with more than a single value, 
        # multiple data values must be packed into an array
        
    # routine has finished normally, passback null error and data value
    else callback null, value 
    
  # routine has caught an exception, passback the exception as an error
  catch exception then callback exception 

module.exports = {start}
```

Usage
=========
*yield from* or *start* a routine iterator
a routine may *yield from* another routine iterator or
a routine may *yield* a promise callback that accepts the standard callback (error, data)->
callback errors are thrown as routine exceptions
```coffeescript
# this is a routine because it yields a callback promise 'yield (callback)->'
sleep = (duration)->
  yield (callback)-> setTimeout (-> callback()), duration

# we can start an anonymous routine iterator because it has 'yield from' a routine iterator inside it
start do ->
  # we can yield from this sleep iterator because it yield to a callback that accepts (error, data...)->
  console.log 'start'
  yield from sleep 1000
  console.log 'slept for 1 sec'
  yield from sleep 1500
  console.log 'slept for 1.5 sec'
  
# routines can be properties of classes
class TestClass
  constructor:(@testValue)->
  test:(value)->
    yield from sleep 1000 # sleep 1 second
    @testValue + value # return to yield
    
routine1 = ->
  yield (callback)-> callback null
  
routine2 = ->
  yield from routine1()
    
testInstance = new TestClass 'Hello'

start do ->
  result1 = yield from routine1()
  result2 = yield from routine2()
  result = yield from testInstance.test ' world!'
  console.log result # logs 'Hello world!'

  testCallbackError = ->
    yield (callback)-> setTimeout (-> callback 'something bad happend durning callback'), 1000
    
  testRoutineError = ->
     yield from sleep 1000
     throw 'something bad happend in routine'

  try
    yield from testCallbackError()
  catch error
    console.error 'Callback Error:', error
    
  try
    yield from testRoutineError()
  catch error
    console.error 'Routine Error:', error
```

Examples
========
Convert async callbacks into routines.
```coffeescript
class FileSystem
  fs = require 'fs'
  
  readFile:(args...)->
    yield (callback)-> fs.readFile args..., callback
    
  writeFile:(args...)->
    yield (callback)-> fs.writeFile args..., callback

coffee = require 'coffee-script'
fs = new FileSystem
start do ->
  sourcedata = yield from fs.readFile 'some/path/to/file.coffee', encoding:'utf8'
  targetdata = coffee.compile sourcedata
  yield from fs.writeFile 'some/path/to/file.js', targetdata, encoding:'utf8'
  console.log 'source file compiled'
```
Use WaitAll to yield to multiple routines.
```coffeescript
compile = (sourcepath, targetpath)->
  sourcedata = yield from fs.readFile sourcepath, encoding:'utf8'
  targetdata = coffee.compile sourcedata
  yield from fs.writeFile targetpath, targetdata, encoding:'utf8'
  sourcepath + ' compiled to ' + targetpath
  
start do ->
  {wait, all} = new WaitAll
  wait compile 'routines.coffee', 'routines.js'
  wait compile 'view.coffee', 'view.js'
  wait compile 'index.coffee', 'index.js'
  yield all # resumes once all 3 waits have resumed
````
Create a route routine for express to eliminate callback spaghetti code and/or function chaining.
```coffeescript
{start} = require 'routines'
express = require 'express'

# declare this just once
route = (routine)->
  (request, response, next)->
    # pass in a error/data handler as second argument to start
    start (routine arguments...), (error, data)->
      if error
        if typeof error == 'string' then next error # send the string error back to the client
        else 
          console.error error.stack # a real exception has occured, just log it and notify the client
          next 'Internal Server Error'
      else response.json data # send any routine return data back as json

router = express()

# use it route here to transform the normal function route into a routine
router.post '/login', route (request, response)->
  {username, password} = JSON.parse yield from request.getData()
  user = yield from db.User.find {username, password}
  throw 'invalid username and password' unless user

  user.sid = generateSID()
  yield from db.User.save user
  
  response.cookies = {sid:user.sid} 
  'password accepted'

# another route routine...
router.get '/profile/:id', route (request)->
  authorize request # authorize session cookie
  user = yield from db.User.find id:request.params.id
  user.profile # send back as json
```
As opposed to this mess...
```coffeescript
router.post '/login', (request, response, next)->
  request.getData (error, data)->
    return next error if error
    {username, password} = JSON.parse data
    
    db.User.find {username, password}, (error, user)->
      return next error if error
      return next 'invalid username and password' unless user

      user.sid = generateSID()
      db.User.save user, (error)->
        return next error if error
        
        response.cookies = user.sid
        response.json 'password accepted'
        
router.get '/profile/:id', (request, response, next)->
  try authorize request
  catch error then return next error
  
  db.User.find id:request.params.id, (error, user)->
    return next error if error
    response.json user.profile
```
