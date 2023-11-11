---
title: "How to build a book application with Golang, Gin and MongoDB."
date: "2022-08-09T23:46:37.121Z"
template: "post"
draft: false
slug: "/posts/how-to-build-a-book-application-with-Golang-Gin-and-MongoDB"
category: "Software Engineering"
tags:
  - "Web Development"
description: "In this article, we'll learn how to build a REST API using Golang, Gin Web Framework and MongoDB."
---

### Introduction
Golang, a simple yet powerful programming language is fast becoming the top choice when building web servers. In this tutorial, I’ll show you how easy it is to build a web application with Golang.  

For this example, we’ll be using [Gin Web Framework](https://github.com/gin-gonic/gin).  Gin is an excellent framework for API development due to its speed and simplicity.


### Target Audience
This article is for anyone with experience programming in Golang and also familiar with building RESTful APIs for their applications. Kindly check out my article on [**Getting started with Golang**](https://kapbyte.netlify.app/posts/getting-started-with-golang/getting-started-with-golang) to get started.


### Prerequisites
To follow along with this tutorial, you will need:

- You need to install Go. For that, visit [the official Go download page](https://go.dev/dl/), and download it for your specific machine.

- A working knowledge of Golang.

- The basics of MongoDB.

- A good understanding of building REST APIs.

### Step 1 — Project Setup

Create a project folder structure

```
mkdir book-app-tutorial
cd book-app-tutorial
```

- Create `go.mod` file by running the command `go mod init github.com/<your-github-username>/book-app-tutorial`

```
mkdir controller database model routes

touch main.go controller/book.controller.go database/database.connection.go model/book.go routes/book.route.go .env
```

- Install packages we'll need for our project.

```
go get -u github.com/gin-gonic/gin
go get go.mongodb.org/mongo-driver/mongo
go get github.com/joho/godotenv
go get github.com/go-playground/validator/v10
```

Our project structure should look as shown below ⬇️

![Project structure](/media/project-folder-structure.png)

### Step 2 — Database and model setup.

- **Step 2.1 :** In `database/database.connection.go`. Now let's implement the MongoDB connection for our application.

If you have issues setting up MongoDB compass locally, kindly check [this](https://www.mongodb.com/docs/compass/current/install/) out.

```
package database

import (
	"context"
	"fmt"
	"log"
	"os"
	"time"

	"github.com/joho/godotenv"
	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"
)

func DBinstance() *mongo.Client {
	err := godotenv.Load(".env")
	if err != nil {
		log.Fatal("Error loading .env file")
	}

	MongoDB := os.Getenv("MONGODB_URL")

	client, err := mongo.NewClient(options.Client().ApplyURI(MongoDB))
	if err != nil {
		log.Fatal(err)
	}

	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	err = client.Connect(ctx)
	if err != nil {
		log.Fatal(err)
	}

	fmt.Println("Connected to MongoDB")
	return client
}

var Client *mongo.Client = DBinstance()

func OpenCollection(client *mongo.Client, collectionName string) *mongo.Collection {
	var collection *mongo.Collection = client.Database("book-DB").Collection(collectionName)
	return collection
}

```

- **Step 2.2 :** In `model/book.go` file, let's create our book schema.

```
package model

import (
	"time"

	"go.mongodb.org/mongo-driver/bson/primitive"
)

type Book struct {
	ID          primitive.ObjectID `bson:"_id"`
	Author      *string            `json:"author" validate:"required"`
	Title       *string            `json:"title" validate:"required"`
	Description *string            `json:"description" validate:"required"`
	Created_at  time.Time          `json:"created_at"`
	Updated_at  time.Time          `json:"updated_at"`
}
```

- **Step 2.3 :** Now let's complete our database configuration by updating our `.env` file.

```
PORT=8080
MONGODB_URL=mongodb://localhost:27017/book-app-db
```

### Step 3 — Implementation of Book APIs.

These are the APIs we are going to build:

| METHOD      | URL                               | ACTION                         |
| :---                |    :----:                          |          ---:                        |
| POST            | /books/create              | Create new book item   |
| GET              | /books/:book_id           | Fetch a book item  |
| PATCH          | /books/:book_id           | Update a book item |
| DELETE        | /books/:book_id           | Delete a book item   |
| GET               | /books                         | Get all books |


- **Step 3.1 :** In `controller/book.controller.go`. Now let's implement our API endpoint to CREATE a new book.

```
package controller

import (
	"context"
	"net/http"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/go-playground/validator/v10"
	"github.com/kapbyte/book-app-tutorial/database"
	"github.com/kapbyte/book-app-tutorial/model"
	"go.mongodb.org/mongo-driver/bson/primitive"
	"go.mongodb.org/mongo-driver/mongo"
)

var bookCollection *mongo.Collection = database.OpenCollection(database.Client, "book")
var validate = validator.New()

func CreateBook() gin.HandlerFunc {
	return func(c *gin.Context) {
		var ctx, cancel = context.WithTimeout(context.Background(), 100*time.Second)
		defer cancel()

		var book model.Book

		if err := c.BindJSON(&book); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}

		validationErr := validate.Struct(book)
		if validationErr != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": validationErr.Error()})
			return
		}

		book.Created_at, _ = time.Parse(time.RFC3339, time.Now().Format(time.RFC3339))
		book.Updated_at, _ = time.Parse(time.RFC3339, time.Now().Format(time.RFC3339))
		book.ID = primitive.NewObjectID()

		result, insertErr := bookCollection.InsertOne(ctx, book)
		if insertErr != nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": "Book item was not created."})
			return
		}

		c.JSON(http.StatusCreated, result)
	}
}
```

- **Step 3.2 :** In `routes/book.route.go`. We'll have all the endpoints routes written out, and comment out the API that has not been built.

```
package routes

import (
	controllers "github.com/kapbyte/book-app-tutorial/controller"

	"github.com/gin-gonic/gin"
)

func BookRoutes(incomingRoutes *gin.Engine) {
	incomingRoutes.POST("books/create", controllers.CreateBook())
	// incomingRoutes.GET("books/:book_id", controllers.GetBook())
	// incomingRoutes.PATCH("books/:book_id", controllers.UpdateBook())
	// incomingRoutes.DELETE("books/:book_id", controllers.DeleteBook())
	// incomingRoutes.GET("books", controllers.GetAllBooks())
}
```

- **Step 3.3 :** Let's import our routes package to our `main.go` file.

```
package main

import (
	"log"
	"os"

	"github.com/joho/godotenv"

	routes "github.com/kapbyte/book-app-tutorial/routes"

	"github.com/gin-gonic/gin"
)

func main() {
	err := godotenv.Load(".env")
	if err != nil {
		log.Fatal("Error loading .env file")
	}

	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}

	router := gin.Default()

	routes.BookRoutes(router)

	router.Run(":" + port)
}

```

Now we start our application by running `go run main.go`. Expected output as shown below ⬇️ 

![Start main.go](/media/start-main.go.png)

Now our server is up and running 😎. We'll be using [Postman](https://www.postman.com/downloads/) to call our APIs across the application. 

![Create new book item](/media/create-book-postman.png)

Now our application works as expected. Let's continue with other endpoints. We would create an API to fetch a book item.

- **Step 3.4 :** Update our `controller/book.controller.go` file by adding the endpoint as shown below ⬇️

```
func GetBook() gin.HandlerFunc {
	return func(c *gin.Context) {
		var ctx, cancel = context.WithTimeout(context.Background(), 100*time.Second)
		defer cancel()

		bookId := c.Param("book_id")
		var book model.Book

		objectId, _ := primitive.ObjectIDFromHex(bookId)

		err := bookCollection.FindOne(ctx, bson.M{"_id": objectId}).Decode(&book)
		if err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": "Error occured while fetching book."})
			return
		}

		c.JSON(http.StatusOK, book)
	}
}
```

➡️ Let's uncomment the `incomingRoutes.GET("books/:book_id", controllers.GetBook())` in our `routes/book.route.go` file, save then restart your server by running `go run main.go`

➡️ Kindly note that you'll need to stop and restart your server, after updating the `routes/book.route.go` file.

![Get book by id](/media/get-book-postman.png)

From the image above 👆🏽, we can now fetch a book item.

- **Step 3.6 :** In `controller/book.controller.go`. Let's work on the endpoint to UPDATE a book item

```
func UpdateBook() gin.HandlerFunc {
	return func(c *gin.Context) {
		var ctx, cancel = context.WithTimeout(context.Background(), 100*time.Second)
		defer cancel()

		bookId := c.Param("book_id")
		var book model.Book

		if err := c.BindJSON(&book); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}

		objectId, _ := primitive.ObjectIDFromHex(bookId)
		filter := bson.M{"_id": objectId}

		var updateObj primitive.D

		if book.Author != nil {
			updateObj = append(updateObj, bson.E{Key: "author", Value: book.Author})
		}

		if book.Title != nil {
			updateObj = append(updateObj, bson.E{Key: "title", Value: book.Title})
		}

		if book.Description != nil {
			updateObj = append(updateObj, bson.E{Key: "description", Value: book.Description})
		}

		book.Updated_at, _ = time.Parse(time.RFC3339, time.Now().Format(time.RFC3339))
		updateObj = append(updateObj, bson.E{Key: "updated_at", Value: book.Updated_at})

		upsert := true
		opt := options.UpdateOptions{
			Upsert: &upsert,
		}

		_, err := bookCollection.UpdateOne(
			ctx,
			filter,
			bson.D{
				{Key: "$set", Value: updateObj},
			},
			&opt,
		)

		if err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": "Book item update failed."})
			return
		}

		c.JSON(http.StatusOK, gin.H{"message": "Book item updated successfully."})
	}
}
```

➡️ Let's uncomment the `incomingRoutes.PATCH("books/:book_id", controllers.UpdateBook())` in our `routes/book.route.go` file, save then restart your server by running `go run main.go`

![Updaate a book item](/media/update-book-postman.png)

- **Step 3.6 :** In `controller/book.controller.go`. Let's work on the endpoint to DELETE a book item.

```
func DeleteBook() gin.HandlerFunc {
	return func(c *gin.Context) {
		var ctx, cancel = context.WithTimeout(context.Background(), 100*time.Second)
		defer cancel()

		bookId := c.Param("book_id")

		objectId, _ := primitive.ObjectIDFromHex(bookId)

		_, err := bookCollection.DeleteOne(ctx, bson.M{"_id": objectId})
		if err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": "Error occured while deleting book item."})
			return
		}

		c.JSON(http.StatusOK, gin.H{"message": "Book item deleted successfully."})
	}
}
```

➡️ Let's uncomment the `incomingRoutes.DELETE("books/:book_id", controllers.DeleteBook())` file, save then restart your server by running `go run main.go`

![Delete book item](/media/delete-book-postman.png)

- **Step 3.6 :** In `controller/book.controller.go`. Let's work on the endpoint to FETCH all books.

```
func GetAllBooks() gin.HandlerFunc {
	return func(c *gin.Context) {
		var ctx, cancel = context.WithTimeout(context.Background(), 100*time.Second)
		defer cancel()

		result, err := bookCollection.Find(context.TODO(), bson.M{})
		if err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": "Error occured while fetching book list"})
			return
		}

		var allBooks []bson.M
		if err := result.All(ctx, &allBooks); err != nil {
			log.Fatal(err)
		}

		c.JSON(http.StatusOK, allBooks)
	}
}
```
➡️ Let's uncomment the `incomingRoutes.GET("books", controllers.GetAllBooks())` file, save then restart your server by running `go run main.go`

![get-books.gif](/media/get-books.gif)

### Conclusion

Congratulations 🥳🎉! You have learned how to build feature-rich applications with Go and the Gin framework.

Please feel free to comment with your thoughts and questions.

The source code for this project is also available on [Github](https://github.com/kapbyte/book-app-tutorial)

Happy coding! 👨🏽‍💻👍🏽