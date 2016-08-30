# ghfhjh
#import "AppDelegate.h"
#import <Parse/Parse.h>
@implementation AppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    [Parse setApplicationId:@"RnBJMkZZRYIi5F9RR3xincVjhc8mYaGr9nMksi5a"

                  clientKey:@"00A6wNGLZcD50t7rGShCiqxXW41yEFeSop3Qifpa"];
                  
    [PFAnalytics trackAppOpenedWithLaunchOptions:launchOptions];
    
    PFObject *testObject = [PFObject objectWithClassName:@"TestObject"];
    
    testObject[@"foo"] = @"bar";
    
    [testObject saveInBackground];
    
    PFObject *supermarket = [PFObject objectWithClassName:@"Supermarket"];
    
    [supermarket setObject:@"apple" forKey:@"fruitItem1"];
    
    supermarket[@"fruitItem2"] = @"orange";
    
    [supermarket saveInBackground];
    
    [self.window makeKeyAndVisible];
    
    
    [PFFacebookUtils initializeFacebook];
    
    if (![PFUser currentUser] && ![PFFacebookUtils isLinkedWithUser:[PFUser currentUser]])
    {
        [self presentLoginControllerAnimated:NO];
    }
    return YES;
}

- (void)presentLoginControllerAnimated:(BOOL)animated {

   
    ParseLoginViewController *loginViewController = [[ParseLoginViewController alloc] init];
    
    loginViewController.delegate = self;
    
    [loginViewController setFields:PFLogInFieldsFacebook];
    
    [self.window.rootViewController presentViewController:loginViewController animated:animated completion:nil];
}

- (BOOL)application:(UIApplication *)application
 
            openURL:(NSURL *)url

  sourceApplication:(NSString *)sourceApplication
  
         annotation:(id)annotation
         {
    return [FBAppCall handleOpenURL:url
    
                  sourceApplication:sourceApplication
                  
                        withSession:[PFFacebookUtils session]];
                        
}

- (void)applicationDidBecomeActive:(UIApplication *)application {

    [FBAppCall handleDidBecomeActiveWithSession:[PFFacebookUtils session]];
    
}

- (void)logInViewController:(PFLogInViewController *)logInController didLogInUser:(PFUser *)user {

    [FBRequestConnection startForMeWithCompletionHandler:^(FBRequestConnection *connection, id result, NSError *error)
    {
        if (!error)
        {
            // handle result
            
            [self facebookRequestDidLoad:result];
            
        }
       
        else {
        
            [self showErrorAndLogout];
        }
    }];
}

- (void)logInViewController:(PFLogInViewController *)logInController didFailToLogInWithError:(NSError *)error {

    // show error and log out
    
    [self showErrorAndLogout];
}

- (void)showErrorAndLogout 
{
    UIAlertView *alertView = [[UIAlertView alloc] initWithTitle:@"Login failed" message:@"Please try again" delegate:nil

cancelButtonTitle:@"OK" otherButtonTitles:nil, nil];

    [alertView show];
    
    [PFUser logOut];
}

- (void)facebookRequestDidLoad:(id)result 
{
    PFUser *user = [PFUser currentUser];

    if (user)
    {
        // update current user with facebook name and id
        
        NSString *facebookName = result[@"name"];
        
        user.username = facebookName;
        
        NSString *facebookId = result[@"id"];
        
        user[@"facebookId"]=facebookId;
        
        // download user profile picture from facebook
        
        
        NSURL *profilePictureURL = [NSURL URLWithString:[NSString
        
        stringWithFormat:@"http://graph.facebook.com/%@/picture?type=square",facebookId]];
        
        NSURLRequest *profilePictureURLRequest = [NSURLRequest requestWithURL:profilePictureURL];
        
        [NSURLConnection connectionWithRequest:profilePictureURLRequest delegate:self];
        
    }
}


- (void)connection:(NSURLConnection *)connection didFailWithError:(NSError *)error
{
    [self showErrorAndLogout];
}

- (void)connection:(NSURLConnection *)connection didReceiveResponse:(NSURLResponse *)response
{
    _profilePictureData = [[NSMutableData alloc] init];
}

- (void)connection:(NSURLConnection *)connection didReceiveData:(NSData *)data 
{
    [self.profilePictureData appendData:data];
}

- (void)connectionDidFinishLoading:(NSURLConnection *)connection 
{
  
  if (self.profilePictureData.length == 0 || !self.profilePictureData)
{
        [self showErrorAndLogout];
    }
    else
{
        PFFile *profilePictureFile = [PFFile fileWithData:self.profilePictureData];

        [profilePictureFile saveInBackgroundWithBlock:^(BOOL succeeded, NSError *error)
        {
            if (!succeeded)
            {
                [self showErrorAndLogout];
            }
            else {
                PFUser *user = [PFUser currentUser];
                
                user[@"profilePicture"] = profilePictureFile;
                
                [user saveInBackgroundWithBlock:^(BOOL succeeded, NSError *error)
                {
                    if (!succeeded)
                    {
                        [self showErrorAndLogout];
                    }
                    else {
                    
                        [self.window.rootViewController dismissViewControllerAnimated:YES completion:nil];
                    }
                }];
            }
        }];
    }
}
// hmvc //

@interface HomeViewController ()

@property (nonatomic, strong) NSMutableArray *followingArray;

@end

@implementation HomeViewController


- (id)initWithCoder:(NSCoder *)aDecoder
{
    self = [super initWithCoder:aDecoder];

    if (self) {
    
        // This table displays items in the Todo class
        
        self.parseClassName = @"Photo";
        
        self.pullToRefreshEnabled = YES;
        
        self.paginationEnabled = YES;
        
        self.objectsPerPage = 3;
    }
    
    return self;
}

- (void)viewDidLoad
{
    [super viewDidLoad];

	// Do any additional setup after loading the view.
	
	
    self.navigationItem.titleView = [[UIImageView alloc] initWithImage:[UIImage imageNamed:@"navbar_logo"]];
}

- (void)viewWillAppear:(BOOL)animated
{
    [super viewWillAppear:animated];

    [self loadObjects];
}


#pragma mark - PFQueryTableViewDataSource and Delegates


- (void)objectsDidLoad:(NSError *)error
{

    [super objectsDidLoad:error];
    
    PFQuery *query = [PFQuery queryWithClassName:@"Activity"];
    
    [query whereKey:@"fromUser" equalTo:[PFUser currentUser]];
    
    [query whereKey:@"type" equalTo:@"follow"];
    
    [query includeKey:@"toUser"];
    
    [query findObjectsInBackgroundWithBlock:^(NSArray *objects, NSError *error)
    {
        if (!error) 
        {
            self.followingArray = [NSMutableArray array];
            
            if (objects.count >0) 
            {
                for (PFObject *activity in objects) 
                {
                    PFUser *user = activity[@"toUser"];
                    
                    [self.followingArray addObject:user.objectId];
                    
                }
            }
            [self.tableView reloadData];
        }
    }];
    
}

// return objects in a different indexpath order. in this case we return object based on the section, not row, the default is row

- (PFObject *)objectAtIndexPath:(NSIndexPath *)indexPath
{
    if (indexPath.section < self.objects.count)
{
        return [self.objects objectAtIndex:indexPath.section];
    }

    else
    {
        return nil;
    }
}

- (UIView *)tableView:(UITableView *)tableView viewForHeaderInSection:(NSInteger)section
{
    if (section == self.objects.count)
{
        return nil;

    }
    
    static NSString *CellIdentifier = @"SectionHeaderCell";
    
    UITableViewCell *sectionHeaderView = [tableView dequeueReusableCellWithIdentifier:CellIdentifier];
    
    PFImageView *profileImageView = (PFImageView *)[sectionHeaderView viewWithTag:1];
    
    profileImageView.layer.cornerRadius = profileImageView.frame.size.width/2;
    
    profileImageView.layer.masksToBounds = YES;
    
    UILabel *userNameLabel = (UILabel *)[sectionHeaderView viewWithTag:2];
    
    UILabel *titleLabel = (UILabel *)[sectionHeaderView viewWithTag:3];
    
    PFObject *photo = [self.objects objectAtIndex:section];
    
    PFUser *user = [photo objectForKey:@"whoTook"];
    
    PFFile *profilePicture = [user objectForKey:@"profilePicture"];
    
    NSString *title = photo[@"title"];
    
    userNameLabel.text = user.username;
    
    titleLabel.text = title;
    
    profileImageView.file = profilePicture;
    
    [profileImageView loadInBackground];
    
    //follow button
    
    FollowButton *followButton = (FollowButton *)[sectionHeaderView viewWithTag:4];
    
    followButton.delegate = self;
    
    followButton.sectionIndex = section;
    
    if (!self.followingArray || [user.objectId isEqualToString:[PFUser currentUser].objectId]) 
    {
        followButton.hidden = YES;
    }
    
    else
    {
        followButton.hidden = NO;
        
        NSInteger indexOfMatchedObject = [self.followingArray indexOfObject:user.objectId];
        
        if (indexOfMatchedObject == NSNotFound)
        {
            followButton.selected = NO;
        }
        
        else 
        {
            followButton.selected = YES;
        }
    }
    
    return sectionHeaderView;
}


- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView 
{
    NSInteger sections = self.objects.count;

    if (self.paginationEnabled && sections >0)
    {
        sections++;
    }
    
    return sections;
}

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section 
{
    return 1;
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath object:(PFObject *)object 
{
    if (indexPath.section == self.objects.count)
{
        UITableViewCell *cell = [self tableView:tableView cellForNextPageAtIndexPath:indexPath];

        return cell;
        
    }
    static NSString *CellIdentifier = @"PhotoCell";
    
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:CellIdentifier];
    
    PFImageView *photo = (PFImageView *)[cell viewWithTag:1];
    
    photo.file = object[@"image"];
    
    [photo loadInBackground];
    
    return cell;
}


- (CGFloat)tableView:(UITableView *)tableView heightForHeaderInSection:(NSInteger)section {
    if (section == self.objects.count)
{
        return 0.0f;
    }

    return 50.0f;
}

- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath 
{
    if (indexPath.section == self.objects.count)
{
        return 50.0f;
    }

    return 320.0f;
}


- (UITableViewCell *)tableView:(UITableView *)tableView cellForNextPageAtIndexPath:(NSIndexPath *)indexPath

{
    static NSString *CellIdentifier = @"LoadMoreCell";
    
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:CellIdentifier];
    
    return cell;
}


- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath
{

    if (indexPath.section == self.objects.count && self.paginationEnabled)
    {
        [self loadNextPage];
    }
}

- (PFQuery *)queryForTable

{
    if (![PFUser currentUser] || ![PFFacebookUtils isLinkedWithUser:[PFUser currentUser]]) 
    {
        return nil;
    }
    
    PFQuery *query = [PFQuery queryWithClassName:self.parseClassName];
    
    [query includeKey:@"whoTook"];
    
    [query orderByDescending:@"createdAt"];
    
    return query;
}

- (void)followButton:(FollowButton *)button didTapWithSectionIndex:(NSInteger)index 

{
    PFObject *photo = [self.objects objectAtIndex:index];
    
    
    PFUser *user = photo[@"whoTook"];
    
    if (!button.selected)
    
    {
        [self followUser:user];
    }
    else 
    {
        [self unfollowUser:user];
    }
    
    [self.tableView reloadData];
}

- (void)followUser:(PFUser *)user

{
    if (![user.objectId isEqualToString:[PFUser currentUser].objectId]) 
    
    {
        [self.followingArray addObject:user.objectId];
        
        
      PFObject *followActivity = [PFObject objectWithClassName:@"Activity"];
      
        followActivity[@"fromUser"] = [PFUser currentUser];
        
        followActivity[@"toUser"] = user;
        
        followActivity[@"type"] = @"follow";
        
        [followActivity saveEventually];
    }
}

- (void)unfollowUser:(PFUser *)user

{
    [self.followingArray removeObject:user.objectId];
    
    PFQuery *query = [PFQuery queryWithClassName:@"Activity"];
    
    [query whereKey:@"fromUser" equalTo:[PFUser currentUser]];
    
    [query whereKey:@"toUser" equalTo:user];
    
    [query whereKey:@"type" equalTo:@"follow"];
    
    [query findObjectsInBackgroundWithBlock:^(NSArray *followActivities, NSError *error)
    
    {
        if (!error)
        {
            for (PFObject *followActivity in followActivities) 
            {
                [followActivity deleteEventually];
            }
        }
    
    }];
    
}



http://stackoverflow.com/questions/7886096/unbalanced-calls-to-begin-end-appearance-transitions-for-uitabbarcontroller-0x   - Unbalanced calls to begin/end appearance transitions for <UITabBarController: 0x197870>














