# Live Project

## Background
As part of my coursework at the Tech Academy, I worked for two weeks in collaboration with a team of fellow students to develop a new website for the local theatre company using ASP .Net MVC and Entity Framework.

## Summary
During my time on this project I was able to work with many aspects of the website including modifying model definitions, altering information in the database and making front-end improvements. When I joined the project there had already been considerable work done so it was a new experience for me to work within an existing codebase. It was a valuable lesson in learning how to explore and learn from what had been done before so as not to duplicate unnecessary work, but also view other’s work with a critical eye so I could offer suggestions to improve the design of the site.

# Back End Stories
## Trash Message Section
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

## Consolidate delete modals
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

## Prevent addition of duplicate images
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
        
## Fix bug when deleted cast member
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
