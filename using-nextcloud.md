# Typical uses of nextcloud in our family

We are using nextcloud to synchronize many things between mobiles, computers and laptops.

First and foremost is file sharing, makes much sense especially when you have an offline password manager like keepass2.

The next is to share calendar and contacts. I also plan to use it to share tasks, i find it very useful to be able to add calendar entries and tasks into the calendar and task lists of other members

Finally, i started to use bookmarks to share browser bookmarks between devices

## Groups and shared folders

File sharing works out of the box for nextcloud, to also have shared folders, do the following:

1. logon to your nextcloud installation as admin
2. click on the admin icon in the top right
3. click on `apps` in the dropdown
4. in category files, look for `group folders` app and install or activate it

Now configure your groups

1. click on the admin icon in the top right
2. click on `user` in the dropdown
3. click on `add group` and add the group name
4. add the group to any user you want to be part of the group

Finally, add your shared group folder

1. click again on the admin icon
2. select `settings` in the drop down
3. fairly down, there should be on the left side an entry `group folders`
4. create a new one and configure it for the group you created above
5. also give `extended rights` to the group so that they can edit the content

This folder is now linked into the synchronized folder of each user of this group. This is great for sharing certain files between family members, while their files in their home folders are only accessable by them.

## File sharing

To use file sharing, you have to install the nextcloud clients on your devices. There are clients available for Android, Windows and Linux.

After installation, start the client and connect to your server. Select where the client should store the synchronized folders.

Done....

## Calendar, contacts and tasks

I tried only two types of devices for calendar, contacts and tasks.

I noticed that it makes sense to structure contacts and calendare entry so that is was easy to share between family members. 

### Contacts

Always split between private address books for your own and public address books you want to share with your family.

I.E. i keep the contact of my local car repair shop in a private address book, same for some other contacts.
First, login as your user and add or import your contacts.

Then click on `settings` on the low left. Add now an address book for each category you need. Then click on the `share` icon, options will expand. Now you can share this address book with your family group. You can also set if the your group can only read it or even change the address book. You can also share with a single person, i.e. share one with your wife only and another with the family group to also share with your kids or parents.

After you have shared an address book with someone or a group, it will also be displayed in this expanded view.

Each user has to do a similar setup if they want to share contacts with someone else or a group

### Calendar

You can share your calendars in a similar way. Switch in the top to `calendar`. Now create new calendars for your important categories. I created a calendar for all anniversaries, one for community garbage collection and one for holidays and school vacations. You can even add a webcal subscription, i added one for my work duty shifts.

For each calendar, there is again the `share` icon. Similar to the contacts, share your calendars either with a person directly or with your family group.

### Android

Android needs some apps installed. You can easily get them from the f-droid market place. For that, go to https://f-droid.org/ and download the apk for `f-droid`. Put that into your shared family folder, you need that on every mobile of your family ;-)

1. To install it on your android device, you have to enable installation from unsecure sources.
2. Install `f-droid`
3. Now open `f-droid` and install `DAVx⁵`,`ICSx⁵` and `Tasks`
4. Open app `DAVx⁵` and add a new account
5. Connect to your nextcloud via URL and user. Use your user
6. Now it will detect all the calendars and address books that are shared with you in your nextcloud
7. You can now select which ones you want to replicate on your phone
8. If you also have a webcal subscription, it will also show them and open `ICSx⁵` to handle the subscription. There is a bug, the checkmark will not be set even if you subscribed successfully.

Now you should see all entries in your calendar app. Depending on your android version and UI modifications, you also can modify view settings in your contacts list to select which address book you want to display in the full list.

If you create a new contact, there is normally an drop down somewhere where you can select to which address book the new contact should be added. Similar is for new calendar entry. 

### Thunderbird

Should probably be the same for Windows and Linux, but i only installed it on Ubuntu Linux.

1. Install `thunderbird`
2. Install add-ons `lightning`, `tbSync` and `DAV-4-TbSync`
3. In `tbSync`. add a new account. Select manual configuration
4. In Nextcloud Browser, go to `settings` on the low left and click on the three dots
5. In the pop-up, click on `copy private link` and paste it into the account settings
6. `tbSync` will now hopefully autodetect all nextcloud resources. Select what you want to replicate

## bookmarks

### Linux Firefox

First, install the bookmarks add-on in nextcloud and enable it. 

I decided to use the `floccus` add-on für firefox. First, create a folder in your bookmark library where you want to keep the bookmarks you want to sync. Then configure your nextcloud account in `floccus`. Its a little bit tricky, every time you leave the pop-up window, all entries are lost until you clicked on save.

After that, bookmarks stored in the synchronized folder will be replicated to nextcloud and any other client device

### Android

For Android, there is an application `nextcloud bookmarks`. Install it, configure your account and booksmarks are synced with nextcloud. To read, push on a bookmark and the app will open up a new tab with the bookmark in the preferred browser.

