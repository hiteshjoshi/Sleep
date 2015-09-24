# Sleep

Sleep is an ODM (**O**bject **D**ocument **M**apper) for **MongoDB** written in Go. It is written on top of the **mgo** library.
Sleep doesn't try to replace mgo, but rather simply augments it.




####Why do I we need an ODM?
You don't. Though, it is nice to have.. specially in the context of a web application. It allows you to couple your business logic with your data. But it is only useful as long as it doesn't get in your way, and you can drop down the to the DB driver any time.

## Features :

*   **Populate** - MongoDB doesn't have JOINs, but we still want to query based on relationships. Sleep makes this task easy. Relationships are mapped via tags in the model definition. Sleep can run populate on a single ObjectId or a slice of them. And it can even run queries on these relationships! For example, lets say you have a social networking site and a user has 200 friends, you can run a query on those friends to only return the friends that are older than 30, sort them by their ages, and limit it to 5 results

*   **Hooks** - The hooks functionality allows you to register functions to be called before or after an action has taken place on the document. Ex: ```PreSave(), PreRemove()```. Use these to consolidate your business logic in one place. See bellow for a full list of supported hooks

*   **Virtuals** - Store computed and temporary data along with your document. These values live only for the lifetime of the document, and are not persisted to the database.

*   **Extends mgo.Collection** - Sleep extends mgo's Collection struct. Reimplements operations that take just a bson.ObjectId to also accept string, because often times we only have a string representation of the ObjectId and we can let sleep handle the conversion. Mgo's Query struct is replaced with one that understands the populate functions.

*   **Convenience methods** - All documents get methods such as `Save(), Remove(), Apply(), Populate(), PopulateQuery()`

* * *

## API Docs
The docs include lots of detailed explainations and examples:

In all of its glory: http://godoc.org/github.com/mansoor-s/Sleep



## Usage: 
### Note:
Sleep suppresses mgo's ErrNotFound error and instead provides the method IsValid() on every document. It returns true if the document was found, otherwise returns false

Define your Model:

```Go
package Models

type User struct {
	Sleep.Document 	`bson:"-"` //This is important! All models must have an anonymous composition of Sleep.Document 	
	Id 		 	bson.ObjectId 		`bson:"_id"`   	//Nothing different from mgo here
	Email 		string
	Password 	string
	Friends 	[]bson.ObjectId 	`model:"User"`  //define relationship - other Users
}

//This is an optional hook implemented to be called when the document is retrieved from the DB
func (u *User) OnResult() {
	u.Virtual.SetInt("totalFriends", len(u.Friends))
}

func (u *User) MySuperDuperMethod() {
	//do cool stuff
}
```
Implementation:

```Go
package main

import (
	"github.com/mansoor-s/Sleep"
	"gopkg.in/mgo.v2"
	"gopkg.in/mgo.v2/bson"
)


func main() {
	//Business as usual.. dial up the DB
	session, err := mgo.Dial("localhost")
	if err != nil {
		panic(err)
	}
	defer session.Close()

	//Create new Sleep instance
	sleep := Sleep.New(session, "MY_DB_NAME")

	//All models must be registered with Sleep
	//It expects an instance of the schema and the collection name that it represents documents in
	// and returns a pointer to a Model representing the mongodb collection
	User := sleep.Register(User{}, "MY_COLLECTION_NAME")

	   /////////////////////////
	  //// Ready to rock!  ////
	 /////////////////////////

	user := &User{}
	//Sleep can infer the model to use from the type of the pointer we passed to Exec()
	//We can pass in both types "string" and "bson.ObjectId"
	err = sleep.FindId("5232171fc081671e81000001").Exec(user)
	if err != nil {
		//do stuff
	}

	if !user.IsValid() {
		//user not found!
	}



	// We can also explictly call Find on the model pointer that we got back from Sleep.Register()
	// Also showing how to handle multiple results
	users := []*User{}
	User.Find(bson.M{"age": 40, "planet": "Earth"}).Sort("firstname", "-lastname").Limit(10).Exec(&users)


	//Using Populate()
	//A populate operation can either be part of a query or can be performed on an existing document

	//////////////////
	//Using Populate In a query:
	//////////////////
	users := []*User{}
	User.Find(bson.M{"age": 40}).Sort("firstname").Limit(10).Populate("Friends").Exec(&users)
	//In this example, lets assume that we got back 10 results. For all of those 10 results, Sleep just populated the references
	//made in its "Friends" field
	
	//To access a populated field:
	theFirstUser := users[0]
	thisUsersFriends := []*User{}
	theFirstUser.Populated("Friends", thisUsersFriends)
	// THAT WAS EASY!!!

	///////////////////
	//Using Populate on an existing document ... say one that you queried from the DB earlier
	//////////////////
	//This example will also show the PopulateQuery method, which allows you to further filter and sort your relationships! 
	friends := []*User{}
	popQuery := User.Find(bson.M{"age": bson.M{"$gt": 30}}).Sort("age").Limit(5)
	err := myDoc.PopulateQuery(popQuery, friends)


	//The model inherits from the mgo.C struct that it represents. For instance, even though Sleep.Model does not implement
	// an EnsureIndex() method, when called, the underlying mgo.C.EnsureIndex() method is called.
	User.EnsureIndex(.....)
	//or
	User.UpdateAll(.....)
	//or
	User.Count()
}
```


--------------------------


###Hooks (Hooks are optional):
```Go
PreSave()
PostSave()
PreRemove()
PostRemove()
OnCreate()
OnResult()
```
Implement these methods in your schema and they will be called when triggered.

Look at the API docs for Sleep.Document for more info



###Virtuals
Example Usage:

```Go
//get the document from DB here..

//Setting virtuals:
myDoc.Virtual.SetString("stringVal", "ABCDEF")

//Store an arbitrary type:
//myIds := []bson.ObjectId{...........}
myDoc.Virtual.Set("myIds", myIds)

//Getting arbitrary typed values
idsInterface, ok := myDoc.Virtual.Get("myIds")
if !ok {
	//handle not found here
}
//assert it back to the type we want
myIds := idsInterface.([]bson.ObjectId)
```