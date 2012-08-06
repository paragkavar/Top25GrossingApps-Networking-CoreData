Top25GrossingApps-Networking-Example
====================================

App that fetches top 25 grossing apps in Apple's Appstore. Features CoreData + Web Services + Multiple Threads

This app is an iPad application that supports all orientation and does NOT use ARC. (All memory management done manually).

It uses CoreData and gets all of its data from web services (http://ax.itunes.apple.com/WebObjects/MZStoreServices.woa/ws/RSS/topgrossingapplications/sf=143441/limit=25/json)

Here is a basic rundown of how the app is architected: 

There is a class called "AppModel" that is the app's model. It is passed an NSManagedObjectContext in the AppDelegate's application:didFinishLaunchingWithOptions: method and does all the core data fetching and saving for us. I find this much cleaner than passing the NSManagedObjectContext to view controllers and having them be responsible for fetching/saving the data. 

Our App object has the following properties: 

	@property (nonatomic, retain) NSString * appCategory;
	@property (nonatomic, retain) NSString * appDescription;
	@property (nonatomic, retain) NSString * applargeImageURLString;
	@property (nonatomic, retain) NSString * appLinkURL;
	@property (nonatomic, retain) NSString * appName;
	@property (nonatomic, retain) NSString * appReleaseDate;
	@property (nonatomic, retain) NSString * appSampleImageURL;
	@property (nonatomic, retain) NSString * appsmallImageURLString;
	@property (nonatomic, retain) NSNumber * appFavorited;
	@property (nonatomic, retain) NSString * appPrice;

AppModel is a a singleton class with an NSManagedObjectContext property: 

	+(AppModel *)sharedInstance {
   	 static AppModel *sharedInstance = nil;
   	 static dispatch_once_t onceQueue;
   	 dispatch_once(&onceQueue, ^{
   	     sharedInstance = [[self alloc] init];
   	 });
   	 return sharedInstance;
	}

	-(AppModel *)init {
 	   self = [super init];
 	   return self;
	}

Before it's implementation file, we define 2 contestants, a backgroundQueue and the URL we will fetch our data from. 

	#define kBgQueue dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)
	#define kAPIURLString @"http://ax.itunes.apple.com/WebObjects/MZStoreServices.woa/ws/RSS/topgrossingapplications/sf=143441/limit=25/json"

It also has the "SaveContext" method 

	- (void)saveContext
	{
   	 NSError *error = nil;
   	 NSManagedObjectContext *managedObjectContext = self.context;
   	 if (managedObjectContext != nil) {
      	  if ([managedObjectContext hasChanges] && ![managedObjectContext save:&error]) {
          	  NSLog(@"Unresolved error %@, %@", error, [error userInfo]);
        	}
  	  }
   	 [error release];
	}

In AppModel.h, we define 2 blocks that we will use to receive the data: 

	typedef void (^CallbackBlock)(NSArray *fetchedAppsArray);
	typedef void (^ErrorBlock)(NSError *error);

Then, in the implementation, we implement the method we will use to fetch or data using these blocks 


	- (void) fetchAppsFromiTunesWithCallback:(CallbackBlock)callbackBlock 	andErrorBlock:(ErrorBlock)errorBlock
	{
 	   __block NSMutableArray *parsedApps = [[NSMutableArray alloc] init];
  	  dispatch_async(kBgQueue, ^{
   	     NSData* data = [NSData dataWithContentsOfURL:[NSURL URLWithString:kAPIURLString]];

   	     NSError* error;
   	     NSDictionary* json = [NSJSONSerialization JSONObjectWithData:data //1
                                                             options:kNilOptions
                                                               error:&error];
        
   	     NSArray* unparsedAppDictionaries = [[json objectForKey:@"feed"] objectForKey:@"entry"];

   	     for (NSDictionary *appDictionary in unparsedAppDictionaries) {
    	        App *app = [App appFromJSONDictionary:appDictionary intoContext:self.context];
   	         [parsedApps addObject:app];
    	    }
     	   if (callbackBlock){
     	       callbackBlock([NSArray arrayWithArray:parsedApps]);
     	   }
   	 });
  	  [self saveContext];
	}

Note the category on the App object, which takes am unparsed NSDictionary + NSManagedObjectContext, and returns an App object. 

	+ (App *)appFromJSONDictionary:(NSDictionary *)appDictionary intoContext:	(NSManagedObjectContext *)context;
	{
  	  App *newApp = [NSEntityDescription insertNewObjectForEntityForName:@"App"
                                         inManagedObjectContext:context];
  	  newApp.appCategory = [[[appDictionary objectForKey:@"category"] objectForKey:@"attributes"] objectForKey:@"label"];
   	 newApp.appDescription = [[appDictionary objectForKey:@"summary"] objectForKey:@"label"];
   	 NSArray *linksArray = [appDictionary objectForKey:@"link"];
   	 newApp.appSampleImageURL = [[[linksArray objectAtIndex:1] objectForKey:@"attributes"] objectForKey:@"href"];
   	 newApp.appLinkURL = [[[linksArray objectAtIndex:0] objectForKey:@"attributes"] objectForKey:@"href"];
    	newApp.applargeImageURLString = [[[appDictionary objectForKey:@"im:image"] objectAtIndex:2] objectForKey:@"label"];
    	newApp.appsmallImageURLString = [[[appDictionary objectForKey:@"im:image"] objectAtIndex:1] objectForKey:@"label"];
    	newApp.appName = [[appDictionary objectForKey:@"title"] objectForKey:@"label"];
    	newApp.appPrice = [[appDictionary objectForKey:@"im:price"] objectForKey:@"label"];    
    	newApp.appReleaseDate = [[[appDictionary objectForKey:@"im:releaseDate"] objectForKey:@"attributes"] objectForKey:@"label"];
    
    	[newApp setAppFavorited:[NSNumber numberWithBool:NO]];
    	return newApp;
    
    	[newApp release];
	}

The AppModel also has methods for favoriting an app and for retrieving all favorited apps: 

	-(NSArray *)favoritedApps
	{
    	NSFetchRequest *favoritesFetch = [NSFetchRequest fetchRequestWithEntityName:@"App"];
   	 [favoritesFetch setPredicate:[NSPredicate predicateWithFormat:@"appFavorited = 1"]];
   	 NSError *error = nil;
   	 NSArray *fetchedFavorites = [self.context executeFetchRequest:favoritesFetch error:&error];
   	 return fetchedFavorites;
	}

	-(void)addAppToFavorites:(App *)app
	{
    	NSFetchRequest *appFetch = [NSFetchRequest fetchRequestWithEntityName:@"App"];
    	[appFetch setPredicate:[NSPredicate predicateWithFormat:@"appName like %@", app.appName]];
   	 NSError *error = nil;
   	 NSArray *fetchedApps = [self.context executeFetchRequest:appFetch error:&error];
   	 App *appToBeFavorited = [fetchedApps lastObject];
   	 [appToBeFavorited setAppFavorited: [NSNumber numberWithBool:YES]];
    	[self saveContext];
	} 

Now for the view controllers. 

The RootViewController has an NSArray property we will use to hold our data when it comes back from the block. 

In rootViewController's viewDidLoadMethod: 

	- (void)viewDidLoad
	{
   	 [super viewDidLoad];
   	 [self showTutorial];
    
   	 isLoading = YES;
    
   	 [[AppModel sharedInstance] fetchAppsFromiTunesWithCallback:^(NSArray *fetchedApps) {
       	 //Set our model
       	 self.appsArray = [NSArray arrayWithArray:fetchedApps];
       	 isLoading = NO;
       	 //Reload tableview on the main thread, as UIKit calls need to be made on the main thread 
       	 dispatch_async(dispatch_get_main_queue(), ^{
        	    [self.tableView reloadData];
        	});
   	 }andErrorBlock:^(NSError *error){
       	 //In a real app, would handle error much more gracefully
       	 NSLog (@"Error found: %@", error);
   	 }];
	}

And in the tableView:cellForRowAtIndexPath: method, 

	- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
	{
   	 if (isLoading) {
      	  static NSString *loadingCellIdentifier = @"LoadingCell";
       	 UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:loadingCellIdentifier];
      	  if (cell==nil) {
        	    cell= [[[UITableViewCell alloc] initWithStyle:UITableViewCellStyleSubtitle reuseIdentifier:loadingCellIdentifier] autorelease];
        	    cell.accessoryType = UITableViewCellAccessoryNone;
       	 }
      	  cell.textLabel.text = @"Loading...";
      	  return cell;
        
    	} else {
    	static NSString *CellIdentifier = @"Cell";
   	 UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:CellIdentifier];
   	 if (cell==nil) {
       	 cell= [[[UITableViewCell alloc] initWithStyle:UITableViewCellStyleSubtitle reuseIdentifier:CellIdentifier] autorelease];
    	    cell.accessoryType = UITableViewCellAccessoryDetailDisclosureButton;
    	}
    	App *app = [self.appsArray objectAtIndex:indexPath.row];
    	cell.textLabel.text = app.appName;
        cell.detailTextLabel.text = [NSString stringWithFormat:@"%@ (%@)", 	app.appCategory, app.appPrice];
    
    	return cell;
   	 }
	}


There is also code in there app for showing detailed views, writing designated initializers, as well as the ability to favorite/unfavorite and see the favorites list in a tableView. 

Make sure to check out the project. 

Thanks for reading! 
