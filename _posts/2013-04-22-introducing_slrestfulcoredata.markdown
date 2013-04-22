---
layout: post
title: Introducing SLRESTfulCoreData
author: oliverletterer
inactive: yes
---

{{ page.title }}
================

Today, I would like to introduce one of our Open Source Frameworks to you: SLRESTfulCoreData. In a nutshell, SLRESTfulCoreData lets you tie a JSON REST API of a webservice easily to your CoreData model. By default SLRESTfulCoreData maps rails conventions (API communicating with JSON objects with underscored attribute names and underscored URLs) into objc conventions. Everything is customizable and integration of other conventions is possible. I think the best approach to explain SLRESTfulCoreData is by example. So let's implement the [GitHub API](http://developer.GitHub.com) in objc _real_ quick:

## Basic setup

The inital project setup is an empty Xcode project with the following dependencies:

* [AFNetworking](https://GitHub.com/AFNetworking/AFNetworking): Everyones favorite networking library which I guess I don't have to introduce here.
* [SLRESTfulCoreData](https://GitHub.com/ebf/SLRESTfulCoreData)
* [CTDataStoreManager](https://GitHub.com/ebf/CTDataStoreManager): An `NSObject` subclass which implements a CoreData stack with 2 `NSManagedObjectContexts`. One main thread context and one background thread context. `CTDataStoreManager` keeps these contexts always in sync by merging changes automatically. A blog post about `CTDataStoreManager` will be coming in the future. The only thing you need to know right now is that we introduce a subclass `GHDataStoreManager` of `CTDataStoreManager` which will manage a CoreData stack for our GitHubAPI and provide the following interface:

```
@interface GHDataStoreManager : CTDataStoreManager

@property (nonatomic, strong) NSManagedObjectContext *mainThreadContext;
@property (nonatomic, strong) NSManagedObjectContext *backgroundThreadContext;

+ (instancetype)sharedInstance;

@end
```

Next, SLRESTfulCoreData needs an object which handles network communication. This has been abstracted with the following protocol:

```
@protocol(SLRESTfulCoreDataBackgroundQueue)

+ (id<SLRESTfulCoreDataBackgroundQueue>)sharedQueue;

- (void)getRequestToURL:(NSURL *)URL
      completionHandler:(void(^)(id JSONObject, NSError *error))completionHandler;

- (void)deleteRequestToURL:(NSURL *)URL
         completionHandler:(void(^)(NSError *error))completionHandler;

- (void)postJSONObject:(id)JSONObject
                 toURL:(NSURL *)URL
     completionHandler:(void(^)(id JSONObject, NSError *error))completionHandler;

- (void)putJSONObject:(id)JSONObject
                toURL:(NSURL *)URL
    completionHandler:(void(^)(id JSONObject, NSError *error))completionHandler;
    
@end
```

This is super fast implemented thanks to AFNetworking in the `GHBackgroundQueue` class. Next we implement an initializer which sets up our SLRESTfulCoreData configuration:

```
__attribute__((constructor))
void GHInitializeSLRESTfulCoreData(void)
{
    @autoreleasepool {
      // register a default background queue
        [NSManagedObject setDefaultBackgroundQueue:[GHBackgroundQueue sharedInstance]];
        
        // register default main and background thread context. SLRESTfulCoreData is designed to not block the main thread in any way. Any changes are therefore performed on the background thread context.
        [NSManagedObject registerDefaultBackgroundThreadManagedObjectContextWithAction:^NSManagedObjectContext *{
            return [GHDataStoreManager sharedInstance].backgroundThreadContext;
        }];
        
        [NSManagedObject registerDefaultMainThreadManagedObjectContextWithAction:^NSManagedObjectContext *{
            return [GHDataStoreManager sharedInstance].mainThreadContext;
        }];
        
        // SLObjectConverter encapsulates conversion between JSON objects and NSManagedObjects. Each NSManagedObject subclass receives its own instance and we can register global default values like:
        [SLObjectConverter setDefaultDateTimeFormat:@"yyyy-MM-dd'T'HH:mm:ss'Z'"];
    }
}
```

The basic setup is done and we can start implementing our first model.

## Implementing the user model

Our first model to implement will be a user. The GitHubAPI returns a specific user from the following route: `GET /users/:user` and returns a JSON object like

```
{
  "login": "octocat",
  "id": 1,
  "avatar_url": "https://GitHub.com/images/error/octocat_happy.gif",
  "gravatar_id": "somehexcode",
  "name": "monalisa octocat",
  "company": "GitHub",
  "blog": "https://GitHub.com/blog",
  "email": "octocat@GitHub.com",
  "created_at": "2008-01-14T04:33:35Z",
  ...
}
```

which we want to store in the following CoreData model:

```
@interface GHUser : NSManagedObject

@property (nonatomic, strong) NSNumber *identifier;
@property (nonatomic, strong) NSString *login;
@property (nonatomic, strong) NSString *avatarURL;
@property (nonatomic, strong) NSString *gravatarIdentifier;
@property (nonatomic, strong) NSString *name;
@property (nonatomic, strong) NSString *company;
@property (nonatomic, strong) NSString *blog;
@property (nonatomic, strong) NSString *email;
@property (nonatomic, strong) NSDate *createdAt;

@end
```

### Attribute mapping

By default, SLRESTfulCoreData maps camelized objc attributes to underscored JSON object attributes. That means `createdAt` will be mapped to `created_at` by default. Therefor we get the mapping for `login`, `name`, `company`, `blog`, `email` and `createdAt` for free. Custom attribute mapping can be done in initializer of `GHUser`:

```
@implementation GHUser

+ (void)initialize
{
    [self registerAttributeMapping:@{
                                     @"identifier": @"id",
                                     @"gravatarIdentifier": @"gravatar_id",
                                     @"avatarURL": @"avatar_url"
                                     }];
}

@end
```

But wait? Is this DRY? GitHub has the convention to give all attributenames containing identifiers an `id` suffix like in `gravatar_id` and all URLs have an `url` suffix. Of course, we would like to introduce some conventions for our own model as well. We would like to map an underscored JSON attribute containg `id` to a camelized objc attribute containing `identifier`. For example, `gravatar_id` should always be mapped to `gravatarIdentifier` and `some_id_value` to `someIdentifierValue`. SLRESTfulCoreData supports these naming conventions and we can extend our global Initilizer as follows:

```
void GHInitializeSLRESTfulCoreData(void)
{
    @autoreleasepool {
        ... 
        
        // SLAttributeMapping encapsulates everything regarding attribute mappings. Each NSManagedObject subclass has with its own instance and we can register global default values here as well:
        [SLAttributeMapping registerDefaultObjcNamingConvention:@"identifier" forJSONNamingConvention:@"id"];
        [SLAttributeMapping registerDefaultObjcNamingConvention:@"URL" forJSONNamingConvention:@"url"];
    }
}
```

Now `gravatar_id` will be automatically mapped to `gravatarIdentifier`, `id` will be mapped to `identifier` and `avatar_url` will be mapped to `avatarURL`. We can delete `+[GHUser initialize]` entirely and nothing more is to be done for `GHUser`'s attribute mapping.

SLRESTfulCoreData automatically type checks all attributes coming from the API to match the types defined in your CoreData model.

### Fetching a single user object

To now fetch a user with a specific name from the GitHub API, we add a class method on `GHUser`:

```
@interface GHUser : NSManagedObject

+ (void)userWithName:(NSString *)name completionHandler:(void(^)(GHUser *user, NSError *error))completionHandler;

@end



@implementation GHUser

+ (void)userWithName:(NSString *)name completionHandler:(void(^)(GHUser *user, NSError *error))completionHandler
{
    NSURL *URL = [NSURL URLWithString:[NSString stringWithFormat:@"/users/%@", name]];
    [self fetchObjectFromURL:URL completionHandler:completionHandler];
}

@end
```

SLRESTfulCoreData introduces a convenience method on NSManagedObject: `+[NSManagedObject fetchObjectFromURL:completionHandler:]` which fetches a JSON object from the specified URL. Since each object has its own unique identifier stored in the `id` attribute, SLRESTfulCoreData will at first try to refresh an existing `GHUser` instance stored in the database. If no such object is found, a new `GHUser` object is being inserted in the database. This has the advantage, that at runtime, only __one__ `GHUser` instance per identifier is in memory (actually two; one on the main thread context and one on the background thread context).

Now fetching a user is as easy as

```
[GHUser userWithName:@"OliverLetterer" completionHandler:^(GHUser *user, NSError *error) {
    NSLog(@"%@", user);
}];
```

and everything regarding the user model is encapsulated in `GHUser` class.

## A second model

Let's setup `GHRepository` real quick:

```
[
  {
    "id": 1296269,
    "owner": {
      "login": "octocat",
      "id": 1,
      "avatar_url": "https://GitHub.com/images/error/octocat_happy.gif",
      "gravatar_id": "somehexcode",
      "url": "https://api.GitHub.com/users/octocat"
    },
    "name": "Hello-World",
    "full_name": "octocat/Hello-World",
    "description": "This your first repo!",
    "private": false,
    "fork": false,
    "url": "https://api.GitHub.com/repos/octocat/Hello-World",
    "html_url": "https://GitHub.com/octocat/Hello-World",
    "clone_url": "https://GitHub.com/octocat/Hello-World.git",
    "git_url": "git://GitHub.com/octocat/Hello-World.git",
    "ssh_url": "git@GitHub.com:octocat/Hello-World.git",
    "svn_url": "https://svn.GitHub.com/octocat/Hello-World",
    "mirror_url": "git://git.example.com/octocat/Hello-World",
    "homepage": "https://GitHub.com",
    "language": null,
    "forks": 9,
    "forks_count": 9,
    "watchers": 80,
    "watchers_count": 80,
    "size": 108,
    "master_branch": "master",
    "open_issues": 0,
    "pushed_at": "2011-01-26T19:06:43Z",
    "created_at": "2011-01-26T19:01:12Z",
    "updated_at": "2011-01-26T19:14:43Z"
  }
]

```

```
@interface GHRepository : NSManagedObject

@property (nonatomic, strong) NSNumber *identifier;
@property (nonatomic, strong) NSString *name;
@property (nonatomic, strong) NSString *fullName;
@property (nonatomic, strong) NSString *repositoryDescription;
@property (nonatomic, strong) NSNumber *private;
@property (nonatomic, strong) NSNumber *fork;
@property (nonatomic, strong) NSString *htmlURL;
@property (nonatomic, strong) NSString *cloneURL;
@property (nonatomic, strong) NSString *gitURL;
@property (nonatomic, strong) NSString *sshURL;
@property (nonatomic, strong) NSString *svnURL;
@property (nonatomic, strong) NSString *mirrorURL;
@property (nonatomic, strong) NSString *homepage;
... 

@property (nonatomic, strong) GHUser *owner;

@end

@implementation GHRepository

+ (void)initialize
{
    [self registerAttributeName:@"repositoryDescription" forJSONObjectKeyPath:@"description"];
}

@end
```

Since `description` has already taken by `NSObject`, we map the JSON `description` attribute to `repositoryDescription` in our own model.

Notice the to-one relationship `owner`? Since the GitHub API returns a user object in `owner`, SLRESTfulCoreData automatically detects the returned data and refreshes an existing `GHUser` object accordingly (if possible). A correct object-graph is automatically created and setup for you. If the API would return an `owner_id` in a repository, the owner relation would be set if an owner with the corresponding `identifier` already exists in the database.

## Fetching repositories of a user

GitHub provides the route `GET /users/:user/repos` for fetching repositories of a given user. SLRESTfuleCoreData extends `NSManagedObject` with `-[NSManagedObject fetchObjectsForRelationship:fromURL:completionHandler:]` for exactly that purpose. Let's first extend the relationship between our `GHUser` and `GHRepository` model on the user site:

```
@interface GHUser : NSManagedObject

...

@property (nonatomic, strong) NSSet *repositories;

@end

@interface GHUser (CoreDataGeneratedAccessors)

- (void)addRepositoriesObject:(GHRepository *)value;
- (void)removeRepositoriesObject:(GHRepository *)value;
- (void)addRepositories:(NSSet *)values;
- (void)removeRepositories:(NSSet *)values;

@end
```

and implement a corresponding method for fetching repositories from that URL:

```
@interface GHUser : NSManagedObject

...

- (void)repositoriesWithCompletionHandler:(void(^)(NSArray *repositories, NSError *error))completionHandler;

@end



@implementation GHUser

...

- (void)repositoriesWithCompletionHandler:(void(^)(NSArray *repositories, NSError *error))completionHandler
{
    NSURL *URL = [NSURL URLWithString:[NSString stringWithFormat:@"/users/%@/repos", self.login]];
    [self fetchObjectsForRelationship:@"repositories" fromURL:URL completionHandler:completionHandler];
}

@end
```

That's it. Let's take a look at the following snippet:

```
[GHUser userWithName:@"OliverLetterer" completionHandler:^(GHUser *user, NSError *error) {
    [user repositoriesWithCompletionHandler:^(NSArray *repositories, NSError *error) {
        
        for (GHRepository *repository in repositories) {
            repository.owner == user; # => true
        }
    }];
}];
```

The owner of each repository in the above example is the same as the user object and can be compared with `==`.

SLRESTfulCoreData provides some really cool techniques to make default tasks like fetching objects and fetching objects for relationsships easier.

* URL substitution:

Building the corresponding URL from which objects will be fetched can be quiet ugly in case you would have to worry about percent escapes and lots of values beeing substituted into the URL. Thanks to SLRESTfulCoreDatas URL substitution feature, we could also specify the URL in the above example as `[NSURL URLWithString:@"/users/:login/repos"]` where `:login` will be evaluated in the context of the current `GHUser` instance. Since URL naming conventions are underscored based, we would specify them here in an underscored way as well. For example the key path `:repository.some_user.gravatar_id` will be translated into the objc key path `repository.someUser.gravatarIdentifier`.

* Generated accessors at runtime

Fetching objects for relationships in such a way seem like a repetitive task as well. Therefore, you can specify CRUD base URLs for each relationship in your initializer like:

```
+ (void)initialize
{
    [self registerCRUDBaseURL:[NSURL URLWithString:@"/users/:login/repos"] forRelationship:@"repositories"];
}
```
. SLRESTfulCoreData can now implement `-[GHUser repositoriesWithCompletionHandler:]` at runtime by implementing its own version of `+[NSManagedObject resolveInstanceMethod:]`. We can remove the implementation `-[GHUser repositoriesWithCompletionHandler:]` entirely and move the definition in the `CoreDataGeneratedAccessors` category:

```
@interface GHUser (CoreDataGeneratedAccessors)

- (void)repositoriesWithCompletionHandler:(void(^)(NSArray *repositories, NSError *error))completionHandler;

@end
```

## CRUD support for models

To demonstrate the last set of features, we are going to implement the issue model. Here is what the GitHub API returns for an issue:

```
[
  {
    "url": "https://api.GitHub.com/repos/octocat/Hello-World/issues/1347",
    "html_url": "https://GitHub.com/octocat/Hello-World/issues/1347",
    "number": 1347,
    "state": "open",
    "title": "Found a bug",
    "body": "I'm having a problem with this.",
    "user": {
      "login": "octocat",
      "id": 1,
      "avatar_url": "https://GitHub.com/images/error/octocat_happy.gif",
      "gravatar_id": "somehexcode",
      "url": "https://api.GitHub.com/users/octocat"
    },
    "labels": [
      {
        "url": "https://api.GitHub.com/repos/octocat/Hello-World/labels/bug",
        "name": "bug",
        "color": "f29513"
      }
    ],
    "assignee": {
      "login": "octocat",
      "id": 1,
      "avatar_url": "https://GitHub.com/images/error/octocat_happy.gif",
      "gravatar_id": "somehexcode",
      "url": "https://api.GitHub.com/users/octocat"
    },
    "milestone": {
      "url": "https://api.GitHub.com/repos/octocat/Hello-World/milestones/1",
      "number": 1,
      "state": "open",
      "title": "v1.0",
      "description": "",
      "creator": {
        "login": "octocat",
        "id": 1,
        "avatar_url": "https://GitHub.com/images/error/octocat_happy.gif",
        "gravatar_id": "somehexcode",
        "url": "https://api.GitHub.com/users/octocat"
      },
      "open_issues": 4,
      "closed_issues": 8,
      "created_at": "2011-04-10T20:09:31Z",
      "due_on": null
    },
    "comments": 0,
    "pull_request": {
      "html_url": "https://GitHub.com/octocat/Hello-World/issues/1347",
      "diff_url": "https://GitHub.com/octocat/Hello-World/issues/1347.diff",
      "patch_url": "https://GitHub.com/octocat/Hello-World/issues/1347.patch"
    },
    "closed_at": null,
    "created_at": "2011-04-22T13:33:48Z",
    "updated_at": "2011-04-22T13:33:48Z"
  }
]
```

For simplicity, we won't implement the labels or milestones model here. Our CoreData model will look like:

```
@interface GHIssue : NSManagedObject

@property (nonatomic, strong) NSNumber *number;
@property (nonatomic, strong) NSString *htmlURL;
@property (nonatomic, strong) NSString *state;
@property (nonatomic, strong) NSString *title;
@property (nonatomic, strong) NSString *body;
@property (nonatomic, strong) NSNumber *comments;
@property (nonatomic, strong) NSDate *closedAt;
@property (nonatomic, strong) NSDate *createdAt;
@property (nonatomic, strong) NSDate *updatedAt;

@property (nonatomic, strong) GHUser *user;
@property (nonatomic, strong) GHUser *assignee;
@property (nonatomic, strong) GHRepository *repository;

@end



@implementation GHIssue

+ (void)initialize
{
    [self registerUniqueIdentifierOfJSONObjects:@"number"];
    
    [self registerCRUDBaseURL:[NSURL URLWithString:@"/repos/:repository.full_name/issues"]];
}

@end
```

Notice two things:

* GitHub is not following its `id` convention so we have to register the new name `number` for the attribute holding the unique key.
* The CRUD base URL for an issue is defined in _underscored_ naming conventions, as explained above.

This will give us multiple things:

* SLRESTfulCoreData implements the following methods at runtime for us:

```
@interface GHRepository (CoreDataGeneratedAccessors)

// GET /repos/:self.full_name/issues
- (void)issuesWithCompletionHandler:(void(^)(NSArray *issues, NSError *error))completionHandler;

// POST /repos/:repository.full_name/issues
- (void)addIssuesObject:(GHIssue *)issue withCompletionHandler:(void(^)(GHIssue *issue, NSError *error))completionHandler;

// DELETE /repos/:repository.full_name/issues/:number
- (void)deleteIssuesObject:(GHIssue *)issue withCompletionHandler:(void(^)(GHIssue *issue, NSError *error))completionHandler;

@end
```

* SLRESTfulCoreData also provides the following CRUD methods for us:

```
GHIssue *issue = [NSEntityDescription insertNewObjectForEntityForName:NSStringFromClass([GHIssue class])
                                               inManagedObjectContext:[GHDataStoreManager sharedInstance].mainThreadContext];
issue.number = @1;
issue.repository = ...;

// GET /repos/:repository.full_name/issues/:number
[issue updateWithCompletionHandler:^(GHIssue *issue, NSError *error) {
  
}];

// POST /repos/:repository.full_name/issues
[issue createWithCompletionHandler:^(GHIssue *issue, NSError *error) {
    
}];

// PATCH /repos/:repository.full_name/issues/:number
[issue saveWithCompletionHandler:^(GHIssue *issue, NSError *error) {
    
}];

// DELETE /repos/:repository.full_name/issues/:number
[issue deleteWithCompletionHandler:^(NSError *error) {
    
}];
```


If you like SLRESTfulCoreData, you can go ahead and take a look at the [Source Code](https://github.com/OliverLetterer/SLRESTfulCoreData), checkout out the [GitHubAPI](https://GitHub.com/OliverLetterer/GitHubAPI) sample project or [follow me on Twitter](https://twitter.com/oletterer).