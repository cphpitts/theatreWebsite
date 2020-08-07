# Live Project

## Background
I worked for two weeks in collaboration with a team of fellow students to develop a new website for the local theatre company using ASP .Net MVC and Entity Framework.

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
