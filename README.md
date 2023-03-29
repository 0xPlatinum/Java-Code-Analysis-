# Java-Code-Analysis!?!?!?


# The Challenge

![image](https://user-images.githubusercontent.com/98354876/228397229-7df57e2a-04b0-4508-8ea7-2302f2baa74c.png)

So! Looks like our goal is to read the "Book Flag" whatever that means. We have some creds to start off, that being user:user, so lets dive in!


# Initial Reconassiance

![image](https://user-images.githubusercontent.com/98354876/228397428-9207a28b-4e9f-484f-bcb6-5ba15791db8f.png)

After logging in with the credentials we were given, it looks like there are 3 books! Aha, the flag book! Well this was easy, lets just read it!

![image](https://user-images.githubusercontent.com/98354876/228397520-922ff59b-e448-4990-b627-baf245ff07f0.png)

Aww, no easy wins, we are blocked from reading the flag book..
The same thing happens when we try to read the Premium book as well. We are only allowed to read the free book.
There is a search function though, So I attempted to do a little SSTI (Server Side Template Injection)

![image](https://user-images.githubusercontent.com/98354876/228397742-b15e9a18-9482-4eb3-85fb-38d2781e41f1.png)

Sadly it failed. I wouldve known if it succeeded if it returned "48" instead of my exact query.

Ok, time to move on with enumeration.
Clicking our little guy in the top right opens up some options

- My profile
- About
- Logout

The about gives some info about the site: Who developed it and what version of the site it is.
The my profile lets us edit our profile!

![image](https://user-images.githubusercontent.com/98354876/228397953-c6c2fb9f-9ca5-49ee-ae7b-e449e41bd789.png)


Clicking on the pencil gives us these options:
![image](https://user-images.githubusercontent.com/98354876/228398077-3ffb9e6e-8d36-47ba-bfa4-f1d02b86a7d9.png)

We can upload a new profile pic, or change the password. We need the current password though, but we dont know anyone elses password.

Ok, time to do the worst thing possible; Analyze java source code.

# Hand me a shovel, because its time to dig through these files

After some time digging through files upon files, I found this in "ReauthenticationFilter.java"

![image](https://user-images.githubusercontent.com/98354876/228399022-6f2ffe6d-2043-4477-9fa6-4a3d227ce4da.png)

Trusting user input, eh? Interesting... Lets keep that in mind, shall we?
What the code is doing from what I can tell is whenever someone tries reauthenticating, it checks if we have a JWT (Json Web Token) and if our security context is null. if it *is* null, it reauthenticates with the users ID and role, then makes a new token.

After some more digging (Theres alot of it) I found this file.

![image](https://user-images.githubusercontent.com/98354876/228399457-58896a33-9de0-4761-89e0-3243710d6c14.png)

Now this code is suppose to check for directory traversal attacks.. but it just doesnt? Well that seems dangerous.. Lets keep that in mind as well, maybe we can override a file on the server with our own!

Next, I decided to checkout the security directory and how the secrets were generated

![image](https://user-images.githubusercontent.com/98354876/228399785-f1133623-7c5e-4470-9cb9-7340a0484623.png)

Interesting! So if the server fails to read the server secret somehow, it generates a new secret of 1234! If this happens we can edit our JWT and become admin!

After reading through another file, UserService.java, we see this snippet of code!
![image](https://user-images.githubusercontent.com/98354876/228400314-310037ca-1297-4f2d-a4a0-5be6eac23096.png)

If you'd please direct your attention to my crappily circled line of code, youd see that any image we upload, the name is never checked for directory traversal! Aha! This *must* be it!

...Except it wasnt. Me and my team spent an hour or two messing with this, attempting to upload an image titled "../../server_secret.txt" to overwrite the secret, force it to generate a new one, and then edit our web token with jwt.io.

alas.. it never worked. So it was back to the source! 
Until it wasn't.
I decided to check out the hints that were given to us for the challenge

![image](https://user-images.githubusercontent.com/98354876/228400700-01320280-b405-4532-bf49-5ea2976c0d83.png)

They specified crack. They **SPECIFIED** CRACK.
Suddenly it all clicked in my head; we had to crack the JWT and use *that* to make a new token.
So, lets take our JWT and use a tool called "hashcat" to crack it with the wordlist rockyou.txt

...COME ON MAN

![image](https://user-images.githubusercontent.com/98354876/228401343-d632ce70-e6b4-425a-9da6-b16905cc1e05.png)

# Getting admin

As you can tell from the screenshot above.. The JWT secret was 1234 *ANYWAY*! We never needed to overwrite it, because it was always that!
Frustrating, but this means we can now edit our JWT with a website i mentioned briefly before, jwt.io

![image](https://user-images.githubusercontent.com/98354876/228401572-dfe5488f-904e-4454-82cf-76e0183d1029.png)

So this is currently what the web token holds. Our current ID as a free user is 1. Lets try to dig through the source one final time to find what the admins information is.
![image](https://user-images.githubusercontent.com/98354876/228401828-47a0b1b5-fc73-4e4c-b136-05c664f5b7e0.png)

Ah there it is! Admin is the third user created, so it must have the role id 3! We can also see its email and full username, lets change our information to match the admins!

![image](https://user-images.githubusercontent.com/98354876/228402078-122b71b9-c354-458f-a374-47ebd8e2bb1a.png)

Note after updating the info, we put the secret in the bottom left, and we have a verified signature! Now, lets copy our new cookie and replace our old one.

![image](https://user-images.githubusercontent.com/98354876/228402210-8a826a8c-08ac-4c98-8cd5-35c40e9571e7.png)

But we are still a free user??
After looking at the cookie.. I noticed the payload below it. Lets change *that* information as well to match our cookie, maybe they need to be the same?

it worked? Maybe??
![image](https://user-images.githubusercontent.com/98354876/228402369-8959b576-6d8e-40a3-ae70-ef2a7c747d33.png)

We clearly messed with *something*, and it says we are admin on the top right..

But trying to load the flag book results in an error

![image](https://user-images.githubusercontent.com/98354876/228402458-01509c75-32f9-4753-852c-d4e6f9cb7e21.png)

Weird..
It was at this point I decided to navigate to my profile again, and thats when I noticed something new!
![image](https://user-images.githubusercontent.com/98354876/228402539-f2152b59-01e1-4e95-9d9b-cf6c9dc79b5f.png)

An admin dashboard!

![image](https://user-images.githubusercontent.com/98354876/228402576-8b9ff503-d9c6-40f9-878e-58fe9e0df17e.png)

Navigating there, we can see the users, books, and an option to add new books! 
Trying to add a book results in an error, sadly, but you know what *doesnt* result in an error?

![image](https://user-images.githubusercontent.com/98354876/228402656-eca7f627-0988-44a9-b677-442e627fa1ed.png)

The editing of a user!
We can edit our original user to be an admin, rather than free!
![image](https://user-images.githubusercontent.com/98354876/228402711-2f27f940-e01b-480a-b20b-93e16772897b.png)

Success!
Now.. lets log out of this borked account and re-login as user, where we should hopefully be admin!

![image](https://user-images.githubusercontent.com/98354876/228403238-dc5cecac-1141-4fec-9c0f-660338fd5bdf.png)

We successfully login as User with Admin permissions! Thus, letting us read the flag and complete the challenge
