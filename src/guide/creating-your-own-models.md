---
title: Creating Your Own Models
type: guide
order: 5
---

## The Task Entity

In this section we will create an example To-Do list application with BootGen. The first thing in our domain model will be a task:

```csharp
[HasTimestamps]
public class Task
{
    public string Title { get; set; }
    public string Description { get; set; }
    public bool IsDone { get; set; }
}
```

The `HasTimestamps` attribute on the `Task` class adds two properties to the model: a `Created` and an `Updated` timestamp.
If this attribute is set, then the default implementation of the `TasksService` will set these timestamps properties as needed.

In the `Configuration.cs` file we add this class to the API as a resource:

```csharp
internal static void AddResources(BootGenApi api)
{
    UserResource = api.AddResource<User>(isReadonly: true, authenticate: true);
    api.AddResource<Task>(authenticate: true, parent: UserResource, parentName: "Owner");
}
```

We have set `UserResource` as parent of `TaskResource`, and so we have created a One-To-Many relationship between `User` and `Task`.
The REST API we will access the To-Do items on the following path: `/users/{userId}/tasks`.
Naming the parent "Owner" means that the `Task` entity class will refer to the user it belongs to as "Owner".

Now users have a list of tasks. Lets change the `User` model to reflect that, by adding an other property:

```csharp
public class User
{
    public string UserName { get; set; }
    public string Email { get; set; }
    [ServerOnly]
    public string PasswordHash { get; set; }
    [Resource]
    public List<Task> Tasks { get; set; }
}
```

The `Resource` attribute on the property means that the task list is not part of the user entity, rather it is a nested resource. If we would run the generator now, we would see that the user entity classes remain unchanged by this modification. The only reason of declaring this property is, that we can extend the database seed. Lets do that by adding some example tasks for the first example user in the `Configuration.cs` file:


```csharp
internal static void AddSeeds(SeedDataStore seedStore)
{
     seedStore.Add(UserResource, new List<User> {
        new User {
            UserName = "Sample User",
            Email = "example@email.com",
            PasswordHash = "AQAAAAEAACcQAAAAEL//UdrNeiFjd0hYeQEBOtAN+OXME8tu8kNMTg4wZUrBSt1/t0Okfs389I82ZaIU2Q==", //password123
            Tasks = new List<Task> {
                new Task {
                    Title = "Buy groceries",
                    Description = "Bread, Milk, Eggs"
                },
                new Task {
                    Title = "Clean up",
                    Description = "Kitchen, Bathroom"
                }
            }
        },
        new User {
            UserName = "Sample User 2",
            Email = "example2@email.com",
            PasswordHash = "AQAAAAEAACcQAAAAENZt+JlnUq5Ukt83M//z8Y/GlXWwYj6d260pmjQEz3Usac29eNfhmZTXHCGVOz70Hg==" //password123
        },
        new User {
            UserName = "Sample User",
            Email = "example3@email.com",
            PasswordHash = "AQAAAAEAACcQAAAAENffyhoiBzkUXycLNzvQOYJJGCXsXw+7U2ZL1ED+kCFCnDmL4yGGQT7Xkr4ZaNV8/A==" //password123
        }
    });
}
```

Alright, lets run the generator! Use the following command in the `Generator` folder:

```sh
dotnet run
```

Check the changes in git! You will see the following:
 * A task entity is created both on the client and the server side.
 * A nested controller is created for the tasks.
 * A service and a service interface is created.
 * The service is registered to the dependency injection container.
 * The `DbContext` was extended with a tasks `DBSet`, and the database seed was extended with our example tasks.
 * The REST API specification is extended in the `restapi.yml` file.
 * The Vuex store is extended with the CRUD operations for the task entity.
 
## Playing With The Application From The Browser Console
 
<p class="tip warning v3-warning"> For this step you will need  Vue.js devtools for [Chrome](https://chrome.google.com/webstore/detail/vuejs-devtools/nhdogjmejiglipccpnnnanhbledajbpd?hl=en) or [Firefox](https://addons.mozilla.org/en-US/firefox/addon/vue-js-devtools/) </p>

### Running the application 
To run the code we first need to initialize the database. If you have already done that, delete the `Migrations` folder and the `web_project.db` SQLite database. Navigate to the `WebProject` folder, then run the following commands:

```sh
rm -rf Migrations
rm web_project.db
dotnet ef migrations add InitialCreate
dotnet ef database update
```
Starting up your project you probably want to do this often. You can use the `resetdb.sh` script (or `resetdb.bat` on Windows) as a shortcut.

Start the server by running the following command:

```sh
dotnet run
```
In a separate terminal window start the client, by running the following command in the `WebProject/ClientApp` folder:

```sh
npm run serve
```

Navigate your browser to http://localhost:8080, where you should see a login screen. Open the developer tools. (This can usually be done by hitting F12 or Ctrl + Shift + I or right-clicking anywhere on the and selecting "Inspect".)

Select the "Vue" tab and click on the "Root" component. This will create a `$vm` variable which allows us to interact with our application from the console.

### Send Login Request

Select the "Console" tab, and send a login request with the following line of JavaScript code:

```javascript
loginResponse = await $vm.$store.dispatch('login', {email: 'example@email.com', password: 'password123'})
```
output:
```json
{
  "jwt": "eyJhbGciOiJIU ... ",
  "user": {
    "id": 1,
    "userName": "Sample User",
    "email": "example@email.com"
  }
}
```
If the application is working correctly, then an object must have appeared on the console with a `user` and the `jwt` property. The value of the `jwt` (JSON Web Token) property is our session token. Let's store this token to be used in the following requests:
```javascript
$vm.$store.commit('setJwt', loginResponse.jwt)
```

### CRUD operations on tasks

Save the logged-in user into a variable:

```javascript
user = loginResponse.user
```

Now that we are successfully logged in, we might query our tasks:

```javascript
await $vm.$store.dispatch('getTasksOfUser', user)
```
output:

```json
[
  {
    "id": 1,
    "title": "Buy groceries",
    "description": "Bread, Milk, Eggs",
    "isDone": false,
    "created": "2020-09-26T17:41:00",
    "updated": "2020-09-26T17:41:00"
  },
  {
    "id": 2,
    "title": "Clean up",
    "description": "Kitchen, Bathroom",
    "isDone": false,
    "created": "2020-09-26T17:41:00",
    "updated": "2020-09-26T17:41:00"
  }
]
```
All queried resources are saved in the Vuex store, `$vm.$store.state.tasks` will contain all previously queried tasks.
Calling

```javascript
$vm.$store.state.tasksOfUser.get(loginResponse.user.id)
```
will return the same list of tasks as above.

#### Add A Task

```javascript
newTask = await $vm.$store.dispatch('addTaskToUser', {user: user, task: {title: "Learn BootGen", description: "bootgen.com"}})
```

output:

```json
{
  "id": 3,
  "title": "Learn BootGen",
  "description": "bootgen.com",
  "isDone": false,
  "created": "2020-09-27T09:59:45.8445397+02:00",
  "updated": "2020-09-27T09:59:45.8445762+02:00"
}
```

#### Update

```javascript
newTask.description = "bootgen.com/guide"
await $vm.$store.dispatch('updateTaskOfUser', {user: user, task: newTask})
```

output:

```json
{
  "id": 3,
  "title": "Learn BootGen",
  "description": "bootgen.com/guide",
  "isDone": false,
  "created": "2020-09-27T09:59:45.8445397+02:00",
  "updated": "2020-09-27T10:04:28.1653196+02:00"
}
```

#### Delete

```javascript
$vm.$store.dispatch('deleteTaskOfUser', {user: user, task: newTask})
```

### Removing redundancy

What we did so far works fine, however, there is some redundancy when calling the update and delete method. Technically it would not be necessary to pass the user parameter, as the server knows already which user a task belongs to. To address this problem, add `Task` as a root resource:

```csharp
internal static void AddResources(BootGenApi api)
{
    UserResource = api.AddResource<User>(isReadonly: true, authenticate: true);
    api.AddResource<Task>(authenticate: true, parent: UserResource, parentName: "Owner");
    api.AddResource<Task>(authenticate: true);
}
```

Run the generator, reset the database (with `resetdb.sh` or `resetdb.bat`) and restart the application. Now you can try the newly introduced CRUD operations:

#### Add A Task

```javascript
await $vm.$store.dispatch('addTask',{title: "Learn BootGen", description: "bootgen.com", ownerId: user.id})
```

output:

```json
{
  "id": 3,
  "title": "Learn BootGen",
  "description": "bootgen.com",
  "isDone": false,
  "created": "2020-09-27T09:59:45.8445397+02:00",
  "updated": "2020-09-27T09:59:45.8445762+02:00"
}
```

#### Update

```javascript
newTask.description = "bootgen.com/guide"
await $vm.$store.dispatch('updateTask', newTask)
```

output:

```json
{
  "id": 3,
  "title": "Learn BootGen",
  "description": "bootgen.com/guide",
  "isDone": false,
  "created": "2020-09-27T09:59:45.8445397+02:00",
  "updated": "2020-09-27T10:04:28.1653196+02:00"
}
```

#### Delete

```javascript
$vm.$store.dispatch('deleteTask', newTask)
```

We have now two sets of CRUD operations for the tasks. Make the nested task resource readonly to simplify it:

```csharp
api.AddResource<Task>(authenticate: true, parent: UserResource, parentName: "Owner", isReadonly: true);
```

## The Tags Entity

We now have a basis for a To-Do list application. But what if we also would like to tag certain tasks to be "important" or "urgent" or "fun"? What if we would also like to let users to define what tags they would like to use? For this reason, let's create a `Tag` class:

```csharp
public class Tag
{
    public string Name { get; set; }
    public string Color { get; set; }
}
```

As we did previously with the User class, add the tags list to the `Task` class:

```csharp
[HasTimestamps]
public class Task
{
    public string Title { get; set; }
    public string Description { get; set; }
    public bool IsDone { get; set; }
    [Resource]
    public List<Tag> Tags { get; set; }
}
```

The relation of the `Task` and the `Tag` class is Many-To-Many. This is specified when registering the resource:

```csharp
internal static void AddResources(BootGenApi api)
{
    UserResource = api.AddResource<User>(isReadonly: true, authenticate: true);
    api.AddResource<Task>(authenticate: true, parent: UserResource, parentName: "Owner", isReadonly: true);
    var taskResource = api.AddResource<Task>(authenticate: true);
    api.AddResource<Tag>(authenticate: true);
    api.AddResource<Tag>(authenticate: true, parent: taskResource, manyToMany: true);
}
```

We have registered `Tag` as resource twice. The first is for maintaining a list of possible tags, the second is for assigning tags to tasks.

Add some sample tags to the database seed, by directly adding them to the tasks of the first user:

```csharp
new User
{
    UserName = "Sample User",
    Email = "example@email.com",
    PasswordHash = "AQAAAAEAACcQAAAAEL//UdrNeiFjd0hYeQEBOtAN+OXME8tu8kNMTg4wZUrBSt1/t0Okfs389I82ZaIU2Q==", //password123
    Tasks = new List<Task> {
         new Task {
             Title = "Buy groceries",
             Description = "Bread, Milk, Eggs",
             Tags = new List<Tag> { new Tag { Name = "Urgent", Color="#ff0000" } }
         },
         new Task {
             Title = "Clean up",
             Description = "Kitchen, Bathroom",
             Tags = new List<Tag> { new Tag { Name = "Important", Color="#ff8800" } }
         }
     }
}
```

Run the generator, reset the database and restart the application.

Run the following commands:

```javascript
loginResponse = await $vm.$store.dispatch('login', {email: 'example@email.com', password: 'password123'})
$vm.$store.commit('setJwt', loginResponse.jwt)
user = loginResponse.user
newTask = await $vm.$store.dispatch('addTask',{title: "Learn BootGen", description: "bootgen.com", ownerId: user.id})
```
#### Get the list of tags

```javascript
await $vm.$store.dispatch('getTags')
```

output:
```json
[
  {
    "id": 1,
    "name": "Urgent",
    "color": "#ff0000"
  },
  {
    "id": 2,
    "name": "Important",
    "color": "#ff8800"
  }
]
```

#### Create a new tag

```javascript
fun = await $vm.$store.dispatch('addTag', { name: 'Fun', color: '#00ff00' })
```

output:
```json
{
  "id": 3,
  "name": "Fun",
  "color": "#00ff00"
}
```

#### Assign the new Tag to a task

```javascript
await $vm.$store.dispatch('addTagToTask', { task: newTask, tag: fun })
```
