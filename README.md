# Theatre Website Code Summary

### Background
As part of my coursework at the Tech Academy, I worked for two weeks in collaboration with a team of fellow students to develop a new website for the local theatre company using ASP .Net MVC and Entity Framework.

### Summary
During my time on this project I was able to work with many aspects of the website including modifying model definitions, altering information in the database and making front-end improvements. When I joined the project there had already been considerable work done so it was a new experience for me to work within an existing codebase. It was a valuable lesson in learning how to explore and learn from what had been done before so as not to duplicate unnecessary work, but also view other’s work with a critical eye so I could offer suggestions to improve the design of the site.

---

## Table of Contents
### Messages Section
- [Trash Message Section](#trash-message-section)
- [Consolidate delete modals](#consolidate-delete-modals)
- [Message Search](#message-search)

### Backend
- [Prevent addition of duplicate images](#prevent-addition-of-duplicate-images)
- [Fix bug when deleted cast member](#Fix-bug-when-deleted-cast-member)

### Frontend
- [Make the footer dynamic](#make-the-footer-dynamic)
- [Random Production Photos on Homepage](#random-production-photos-on-homepage)

---

## Messages Page
### Trash Message Section
One of the features of the website is a messaging system to allow users to send and receive messages. When I joined the project an inbox and sentbox were already enabled and I was tasked with developing the deleted messages section.

The first thing I did was change the behavior of the delete method to make use of two new attributes to the message model, SenderDeleted and RecipientDeleted, filling in those attributes with the time of deletion.

    public ActionResult DeleteConfirmed(int id)
    {
      Message message = db.Messages.Find(id);
      var userManager = new UserManager<ApplicationUser>(new UserStore<ApplicationUser>(db));
      ApplicationUser currentUser = userManager.FindById(User.Identity.GetUserId());

      string currentId = currentUser.Id.ToString();

      if (message.SenderId == currentId)
      {
        message.SenderDeleted = DateTime.Now;
      }
      if (message.RecipientId == currentUser.Id)
      {
        message.RecipientDeleted = DateTime.Now;
      }

      db.SaveChanges();
      return RedirectToAction("Index");
    }

I then modified of existing code on the view to create a new tab that would only show the deleted message

    <tbody>
      @foreach (var item in Model.Where(i => (i.SenderId == (string)ViewData["currentUserId"] && i.SenderDeleted != null) || (i.RecipientId == (string)ViewData["currentUserId"] && i.RecipientDeleted != null)).OrderByDescending(i => i.SenderId == (string)ViewData["currentUserId"] ? i.SenderDeleted : i.RecipientDeleted))
      {
        <tr class="d-flex trashRow" id="trash_@item.MessageId" data-msg_id="@item.MessageId">
          <td class="col-2 msgTo ">
            @if (item.Recipient != null)
              {
                @: @item.Recipient.LastName, @item.Recipient.FirstName
              }
          </td>

          <td class="col-2 msgFrom ">
            @if (item.Recipient != null)
            {
              @: @item.Sender.LastName, @item.Sender.FirstName
            }
          </td>

          <td class="col-3 msgSubject ">
            @Html.DisplayFor(modelItem => item.Subject)
          </td>
          <td class="col-2 d-none msgBody ">
            @Html.DisplayFor(modelItem => item.Body)
          </td>
          <td class="col-3 msgDate ">
            @if (item.SenderId == (string)ViewData["currentUserId"])
            {
                @Html.DisplayFor(modelItem => item.SenderDeleted)
            }
            else
            {
                @Html.DisplayFor(modelItem => item.RecipientDeleted)
            }

          </td>
          <td class="col-2 col-lg-2 text-right">
            @* Recover Deleted Message *@
            <span class="badge badge-pill badge-success listRecoverMsg" data-toggle="modal" data-target="#recoverMessageMaster">
                Recover
            </span>

            @* Permanently Delete Message *@
            <span class="badge badge-pill badge-danger listPermDel" data-toggle="modal" data-target="#permanentDeleteMaster">
                <i class="fa fa-trash" aria-hidden="true"></i>
            </span>
          </td>
        </tr>
      }
    </tbody>

I also implemented two new methods for methods, one to recover deleted messages and one two permanently delete the message from the users message box.

To recover a message, I deleted the time from the appropriate attribute (SenderDeleted or RecipientDeleted) and reset the value to null.
    
    [HttpPost, ActionName("Recover")]
    [ValidateAntiForgeryToken]
    public ActionResult RecoverConfirmed(int id)
    {
        Message message = db.Messages.Find(id);
        //db.Messages.Remove(message);
        var userManager = new UserManager<ApplicationUser>(new UserStore<ApplicationUser>(db));
        ApplicationUser currentUser = userManager.FindById(User.Identity.GetUserId());

        string currentId = currentUser.Id.ToString();

        if (message.SenderId == currentId)
        {
            message.SenderDeleted = null;
        }
        if (message.RecipientId == currentUser.Id)
        {
            message.RecipientDeleted = null;
        }

        db.SaveChanges();
        return RedirectToAction("Index");
    }

  To permanently delete a message I added two additional attributes to the message model, SenderPermanentDelete and RecipientPermanentDelete. Messages with a value in these columns would be filtered out of all views for the user. If both sender and recipient marked a message as permanently deleted, then the message would be removed from the database.
   
    [HttpPost, ActionName("PermDel")]
    [ValidateAntiForgeryToken]
    public ActionResult PermDelConfirmed(int id)
    {
        Message message = db.Messages.Find(id);

        var userManager = new UserManager<ApplicationUser>(new UserStore<ApplicationUser>(db));
        ApplicationUser currentUser = userManager.FindById(User.Identity.GetUserId());

        string currentId = currentUser.Id.ToString();

        if (message.SenderId == currentId)
        {
            message.SenderPermanentDelete = true;
        }
        if (message.RecipientId == currentUser.Id)
        {
            message.RecipientPermanentDelete = true;
        }

        if (message.SenderPermanentDelete && message.RecipientPermanentDelete)
        {
            db.Messages.Remove(message);
        }


        db.SaveChanges();
        return RedirectToAction("Index");
    }

### Consolidate delete modals
When deleting a message a modal would appear to confirm the action. When I joined the project this modal was being generating within the foreach loop that generated the list of messages. This resulted in multiple modals being generated on the page. In addition, a separate modal was also coded that handled deletions from the message detail view. I decided to consolidate these into a single modal for each potential action (delete, multiple delte, permanently delete and recover). This great increased the efficiency of the page as it no longer had to generate the code for potentially dozens of modals. I created a jquery function that would populate the data of these modals either when the detail view was loaded or the appropriate button was clicked on the list view.

Below is the function to populate the Delete Confirmation modal. All modals had a similar function.
   
     function populateDelMsg(msgId) {
        var selectedEntryId = "#msg_" + msgId;
        console.log(selectedEntryId + " " + msgId);
        $("#deleteMessageMaster .delMsg").attr("msg_id", msgId);
        if ($(selectedEntryId).find(".msgFrom").length) {
            $("#deleteMessageMaster .del_sender").html($(selectedEntryId).find(".msgFrom").html());
        } else {
            $("#deleteMessageMaster .del_sender").html(@Html.Raw(Json.Encode(ViewData["currentUserName"])));
        }
        if ($(selectedEntryId).find(".msgTo").length) {
            $("#deleteMessageMaster .del_recipient").html($(selectedEntryId).find(".msgTo").html());
        } else {
            $("#deleteMessageMaster .del_recipient").html(@Html.Raw(Json.Encode(ViewData["currentUserName"])));
        }
        $("#deleteMessageMaster .del_time").html($(selectedEntryId).find(".msgDate").html());
        $("#deleteMessageMaster .del_subject").html($(selectedEntryId).find(".msgSubject").html());
      }

### Message Search
I was also tasked with adding a search feature to the messages section. This would allow the user to enter a search query and find all messages with the requested query in their subject, body or sender attributes. First I added the required input onto the messages template

        @* Search Form *@
        <li class="nav-item searchBar mr-auto">
            @using (Html.BeginForm("Index", "Messages", FormMethod.Post))
            {
                @Html.AntiForgeryToken()
                <div class="input-group">
                    <input type="text" class="form-control m-0" name="searchQuery" id="searchQuery" placeholder="Search" aria-label="Search" aria-describedby="basic-addon2">
                    <div class="input-group-append">
                        <button class="btn btn-outline-secondary fa fa-search" type="submit"></button>
                    </div>
                </div>
            }
        </li>
        
I then modified the Index method to accept an string argument. If the string was not empty or null, then the method would find all matching methods. In the messages model, the sender and reciever are stored as ids, so I also had to search the table of users to find the ids of any user that matched the search query and then used those values to match against the sender attribute of the message.
   
        public ActionResult Index(string searchQuery)
        {
            var userManager = new UserManager<ApplicationUser>(new UserStore<ApplicationUser>(db));
            ApplicationUser currentUser = userManager.FindById(User.Identity.GetUserId());
            var filteredUsers = db.Users.OrderBy(i => i.LastName).ThenBy(i => i.FirstName);
            if (!User.IsInRole("Admin"))
            {
                filteredUsers = (IOrderedQueryable<ApplicationUser>)filteredUsers.Where(i => i.Role == "Admin");
            }
            var UserList = filteredUsers.Select(i => new SelectListItem
            {
                Value = i.Id,
                Text = i.LastName + ", " + i.FirstName
            });
            ViewData["listOfUsers"] = new SelectList(UserList, "Value", "Text");
            ViewData["currentUserId"] = currentUser.Id;
            ViewData["currentUserName"] = currentUser.LastName + ", " + currentUser.FirstName;

            // Get all messages for the user
            var messages = db.Messages.Where(i => (i.SenderId == currentUser.Id && i.SenderPermanentDelete != true) || (i.RecipientId == currentUser.Id && i.RecipientPermanentDelete != true)).OrderByDescending(i => i.SentTime).ToList();

            // If there is a search query, filter messages that match the search
            if (!string.IsNullOrEmpty(searchQuery))
            {
                var users = db.Users.Where(u => u.FirstName.Contains(searchQuery) || u.LastName.Contains(searchQuery)).Select(u => u.Id).ToArray();
                messages = messages.Where(m => m.Subject.ToLower().Contains(searchQuery.ToLower()) || m.Body.ToLower().Contains(searchQuery.ToLower()) || users.Any(u => u == m.SenderId) || users.Any(u => u == m.RecipientId)).ToList();
                ViewData["searchQuery"] = searchQuery;
            }
            return View(messages);
        }

---

## Backend
### Prevent addition of duplicate images
Administrators on the site are able to upload pictures for cast members and productions. All of these utilize the same method to create and upload the photo data. I modified the method to search the existing photos for matching data when a user uploaded an image. If no match was found then the new photo was added to the database, otherwise the id of the current photo was returned instead to be associated with the entry.

        [AllowAnonymous]
        public static int CreatePhoto(HttpPostedFileBase file, string title)

        {
            var photo = new Photo();
            using (ApplicationDbContext db = new ApplicationDbContext())
            {
                byte[] photoArray = ImageBytes(file);
                if (db.Photo.Where(x => x.PhotoFile == photoArray).ToList().Any())
                {
                    var id = db.Photo.Where(x => x.PhotoFile == photoArray).ToList().FirstOrDefault().PhotoId;
                    return id;
                }
                else
                {
                    photo.Title = title;
                    Image image = Image.FromStream(file.InputStream, true, true);
                    photo.OriginalHeight = image.Height;
                    photo.OriginalWidth = image.Width;
                    var converter = new ImageConverter();
                    photo.PhotoFile = (byte[])converter.ConvertTo(image, typeof(byte[]));
                    db.Photo.Add(photo);
                    db.SaveChanges();
                    return photo.PhotoId;
                }
            }
        }
        
### Fix bug when deleted cast member
There was an existing bug that would return an error when a castmember was deleted. I investigated the issue and found that the error was caused when there were parts associated with that cast member. To fix this I added a routine to the delete cast member function that searched the table of Parts and found any which the cast memeber was assigned. If matches were found, the value of that reference was changed to null.

        [HttpPost, ActionName("Delete")]
        [ValidateAntiForgeryToken]
        [TheatreAuthorize(Roles = "Admin")]

        public ActionResult DeleteConfirmed(int? id)
        {
            CastMember castMember = db.CastMembers.Find(id);

            // Before the cast member is removed.  Set the associated User CastMemberUserId to 0 if a User was assigned.
            if (castMember.CastMemberPersonID != null)
                db.Users.Find(castMember.CastMemberPersonID).CastMemberUserID = 0;

            // Before cast memeber is removed, search for associated parts and set Person_CastMemberId to null
            foreach (var i in db.Parts.Where(p => p.Person.CastMemberID == id))
            {
                i.Person = null;
            }

            db.CastMembers.Remove(castMember);

            db.SaveChanges();
            return RedirectToAction("Index");
        }
        
In addition I also added some logic to the view files for the production details. There was a section on the page that listed the parts and cast members. I changed this section to not attempt the display the cast member if the value was null.

    <li class="list-group-item">
                  @(part)s
                  <dl>
                    @foreach (var item in parts)
                    {
                        if (item != null && part.ToString() == "Actor")
                        {
                        <dd class="col">
                          <a href="/Part/Details/@item.PartID">@Html.Raw(@item.Character)</a>
                        </dd>
                        if (item.Person != null) { 
                            <dd class="col">
                              <p class="prod-details-actor-info"> played by <a class="prod-details-actor-info-link" href="/CastMembers/Details/@item.Person.CastMemberID">@item.Person.Name.ToString()</a></p>
                            </dd>
                        }
                      }
                      else if (item != null)
                      {
                        <dd class="col">
                          <a href="/Part/Details/@item.PartID">@Html.Raw(@item.Character)</a>
                        </dd>
                      }
                    }
                  </dl>
                </li>

---

## Frontend
### Make the footer dynamic
The client required a way to easily edit information in the footer, including address and phone number. To accomplish this I added a property to an existing JSON file, AdminSettings.JSON.

    "FooterInfo": {
        "AddressStreet": "2110 SE 10TH AVE",
        "AddressCityStateZip": "PORTLAND OR 97214",
        "PhoneSales": "(503) 482-8655",
        "PhoneGeneral": "(503) 482-8655"
    }
    
And I added the corresponding properties into the AdminSettings model

    public Footer FooterInfo { get; set; }
    public class Footer
    {
        public string AddressStreet { get; set; }   // Street address
        public string AddressCityStateZip { get; set; }     // City, State, Zip Address
        public string PhoneSales { get; set; }  // Sales Phone Number
        public string PhoneGeneral { get; set; }    // General Phone Number
    }
        
Next I added the appropriate fields to the admin dashboard.

    @* Inputs for Footer Information *@
    FOOTER INFORMATION<br />
    Address - Street<br />
    <input type="text" name="FooterInfo.AddressStreet" value="@Html.DisplayFor(model => model.FooterInfo.AddressStreet)">
    Address - City, State, Zip<br />
    <input type="text" name="FooterInfo.AddressCityStateZip" value="@Html.DisplayFor(model => model.FooterInfo.AddressCityStateZip)">
    Phone Number - Sales<br />
    <input type="text" name="FooterInfo.PhoneSales" value="@Html.DisplayFor(model => model.FooterInfo.PhoneSales)">
    Phone Number - General<br />
    <input type="text" name="FooterInfo.PhoneGeneral" value="@Html.DisplayFor(model => model.FooterInfo.PhoneGeneral)">
    
Finally I edited the shared Layout.cshtml file to call a helper method to retrieve the relevant information

    @* Get footer information from AdminSettings *@
    @{ var footerContent = TheatreCMS.Helpers.AdminSettingsReader.CurrentSettings().FooterInfo; }

    <div class="float-lg-left text-left" id="contactInfo">
        @*bootstrap classes that align text*@
        <div id="address">
            <p id="addressPadding">
                @*Added for css styling*@
                @footerContent.AddressStreet <br />
                @footerContent.AddressCityStateZip
            </p>
        </div>
        <div id="phoneNumbers">
            <p id="phoneNumbersPadding">
                @*Added for css styling*@
                Phone Number (General) - @footerContent.PhoneGeneral<br />
                Phone Number (Sales) - @footerContent.PhoneSales  
            </p>
        </div>
    </div>
    <div class="float-lg-right text-left mt-lg-4">
        @*Boot strap text aligning*@
        <p class="my-0">&copy; @DateTime.Now.Year</p>
    </div>
    
### Random Production Photos on Homepage
A number of the productions have a number of photos associated with them. The client wanted to display a random image for each production when the homepage was loaded. To accomplish this I set up a script to create a list of photos belonging to the production and selecting the id of a random image. The id was then passed to the standard DisplayPhoto method.

    @{  
        // Establish Random object for displaying random production image
        Random rand = new Random();
        // Open connection to database
        ApplicationDbContext db = new ApplicationDbContext();
    }
    @foreach (var item in Model)
    {
        <div id="index-block-@i" class="home-block">
            <figure class="home-inner">
                <div class="home-tint-overlay"></div>
                @{
                    if (item.DefaultPhoto != null)
                    {
                         // Create list of photos associated with production
                        var photoArray = db.ProductionPhotos.Where(p => p.Production.ProductionId == item.ProductionId).ToList();
                        // Get length of list and choose random number between 0 and length of list
                        var photoCount = photoArray.Count();
                        var randNum = rand.Next(photoCount);
                        // Get id of the selected photo
                        var selectedPhoto = photoArray[randNum].PhotoId;

                        <div class="home-img">
                           <div role="presentation" style="background-image: url(@Url.Action("DisplayPhoto", "Photo", new { id = selectedPhoto }))"></div>
                        </div>
                        <div class="home-mobile-img">
                            <div role="presentation" style="background-image: url(@Url.Action("DisplayPhoto", "Photo", new { id = selectedPhoto }))"></div>
                        </div>
                        ...
