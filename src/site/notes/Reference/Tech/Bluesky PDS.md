---
{"dg-publish":true,"dg-path":"Tech/Bluesky PDS.md","permalink":"/tech/bluesky-pds/","noteIcon":"3"}
---

#tech #selfhost #web 

[Bluesky](https://bsky.app/) is built on the open [ATProto](https://atproto.com/) network. The **P**ersonal **D**ata **S**erver is the network unit that serves as a repository for user data, identity, and media on the network.

<center><blockquote class="bluesky-embed" data-bluesky-uri="at://did:plc:w4xbfzo7kqfes5zb7r6qv3rw/app.bsky.feed.post/3ludcalt7i224" data-bluesky-cid="bafyreigkq2j5c4pov7uqzvyijctafsyroyb4ws63hug2djsz7x267s2zva" data-bluesky-embed-color-mode="dark"><p lang="en">I would just like to inject in everyone&#x27;s imaginations that at least one (1) path to a network of many networks, a fediverse of many fediverses, is a community of many communities. And that might mean more people should found new communities. And tools should be developed to make that easier.</p>&mdash; Rudy wants revolution. (<a href="https://bsky.app/profile/did:plc:w4xbfzo7kqfes5zb7r6qv3rw?ref_src=embed">@rudyfraser.com</a>) <a href="https://bsky.app/profile/did:plc:w4xbfzo7kqfes5zb7r6qv3rw/post/3ludcalt7i224?ref_src=embed">Jul 19, 2025 at 8:39 AM</a></blockquote><script async src="https://embed.bsky.app/static/embed.js" charset="utf-8"></script></center>

### Resources
* [PDS on Github](https://github.com/bluesky-social/pds) - the software to run your own PDS
* [pdsls.dev](https://pdsls.dev/) - handy tool for browsing your PDS
* [pdsadmin-web](https://mkizka.github.io/pdsadmin-web/) - a web based tool for performing some basic admin functions
* [PDS Moover](https://pdsmoover.com/) - migrate accounts between PDS
* [Independent PDS Directory](https://blue.mackuba.eu/directory/pdses)
# Setting up the PDS
Running the official PDS Docker container is pretty simple. It's pretty lightweight, so you don't need a fancy server to run it. I use the OCI Free Tier for both my server and their object storage (S3). You could definitely do this on something like a [Raspberry Pi](https://amzn.to/4mNYSNL).

Before starting this process, you'll want to [set up DNS records](https://github.com/bluesky-social/pds?tab=readme-ov-file#configure-dns-for-your-domain) with your domain provider and give them some time to propagate. 

The Github repo has a script you can use to set all this up automatically. You probably just want to do that!

I already have a server and proxy running with some other software, so I just need to add the PDS to my existing Docker Compose stack.

The relevant portion of my Compose file:
```yaml
  pds:
    container_name: pds
    image: ghcr.io/bluesky-social/pds:latest
    networks:
      - public-proxy
    restart: unless-stopped
    volumes:
      - /home/ubuntu/pds/data:/pds
    env_file:
      - /home/ubuntu/pds/pds.env
```

Running the container will create a template pds.env file at that location specified, which you can then fill out with your secrets and SMTP credentials.
## OCI S3
I also set up S3 storage, which [Bayley Townsend's howto](https://baileytownsend.dev/articles/s3-blob-store) was helpful for figuring out (also the [OCI S3 compatibility](https://docs.oracle.com/en-us/iaas/Content/Object/Tasks/s3compatibleapi.htm) doc). Doing so stores the media in the cloud, so our server's disk space doesn't get eaten up with user content like photos or videos.

Getting set up with OCI S3 was, of course, not that straightforward:
1. [Make note of your **Region Identifier**](https://docs.oracle.com/en-us/iaas/Content/General/Concepts/regions.htm) ex: `us-sanjose-1`
2. [Make a new Object Storage bucket](https://cloud.oracle.com/object-storage) on your account.
3. On **Bucket Details** page, change visibility to *Public*
4. Make note of the **Namespace** on the *Bucket Details* page
5. Go to your *User Profile -> User Settings*. Select **Tokens and Keys**
6. Scroll down to **Customer secret keys**
7. **Generate a Secret key** it will only be shown once
8. Back on the previous page, you now need to copy the **Access key** from `...`

Now we have everything we need to add our secrets to our pds.env file:
```YAML
PDS_BLOBSTORE_S3_BUCKET=<bucket name>
PDS_BLOBSTORE_S3_REGION=<region>
PDS_BLOBSTORE_S3_ENDPOINT=https://<namespace>.objectstorage.<region>.oraclecloud.com
PDS_BLOBSTORE_S3_ACCESS_KEY_ID=<accesskey>
PDS_BLOBSTORE_S3_SECRET_ACCESS_KEY=<secretkey>
PDS_BLOBSTORE_S3_FORCE_PATH_STYLE=true
```

Once you have done so and started the service you'll see this message in your browser:
```
         __                         __
        /\ \__                     /\ \__
    __  \ \ ,_\  _____   _ __   ___\ \ ,_\   ___
  /'__'\ \ \ \/ /\ '__'\/\''__\/ __'\ \ \/  / __'\
 /\ \L\.\_\ \ \_\ \ \L\ \ \ \//\ \L\ \ \ \_/\ \L\ \
 \ \__/.\_\\ \__\\ \ ,__/\ \_\\ \____/\ \__\ \____/
  \/__/\/_/ \/__/ \ \ \/  \/_/ \/___/  \/__/\/___/
                   \ \_\
                    \/_/

This is an AT Protocol Personal Data Server (aka, an atproto PDS)...
```

You should now be able to browse your PDS at it's URL with pdsls. [Here's mine](https://pdsls.dev/hermitary.brad.quest). If you haven't set up an account though there won't be any records.
## Creating Accounts
The PDS container doesn't include pdsadmin tool, so we'll need another way to make invite codes. You can use [pdsadmin-web](https://mkizka.github.io/pdsadmin-web/) to do so easily. Before I was aware of that tool, I did an API call via [Postman](https://postman.co):

* **HTTP Request**
	* **Type:** POST
	* **URL:** `your.pds.tld/xrpc/com.atproto.server.createInviteCode`
	* **Body:** raw `{"useCount":1}`
	* **Auth:** Basic Auth
		* **User:** admin
		* **Pass:** from your pds.env

Once you have the code, you can go to the Bluesky app, enter your PDS and invite, and create an account as normal. Now, moment of truth, make a test post with some media:

<center><blockquote class="bluesky-embed" data-bluesky-uri="at://did:plc:nkurkqfdnnrplphhjnuksdzl/app.bsky.feed.post/3lwhrb2tedk2z" data-bluesky-cid="bafyreigure4da6zirxcercgp4f7hsufymdqs4vlh4d654ibzpfyddlypsy" data-bluesky-embed-color-mode="dark"><p lang="en">it is me<br><br><a href="https://bsky.app/profile/did:plc:nkurkqfdnnrplphhjnuksdzl/post/3lwhrb2tedk2z?ref_src=embed">[image or embed]</a></p>&mdash; scribe.brad.quest (<a href="https://bsky.app/profile/did:plc:nkurkqfdnnrplphhjnuksdzl?ref_src=embed">@scribe.brad.quest</a>) <a href="https://bsky.app/profile/did:plc:nkurkqfdnnrplphhjnuksdzl/post/3lwhrb2tedk2z?ref_src=embed">Aug 15, 2025 at 2:09 PM</a></blockquote><script async src="https://embed.bsky.app/static/embed.js" charset="utf-8"></script></center>

You'll get a `500` error if something goes wrong with uploading an image, meaning S3 isn't working. Double check values in the pds.env and permissions in your OCI account.
## Migrating Accounts
I used [PDS MOOver](https://pdsmoover.com/) and it worked perfectly for consolidating 4 accounts on my new PDS. I migrated from Bluesky servers and my own old server. I yolo'ed it and did not follow [the precautions](https://pdsmoover.com/info.html#precautions), although that seems like a good idea. The process went like this:

<center><blockquote class="bluesky-embed" data-bluesky-uri="at://did:plc:g7j6qok5us4hjqlwjxwrrkjm/app.bsky.feed.post/3lw3hcuojck2u" data-bluesky-cid="bafyreieowrhsk4r2qzx7vknkuvavz5edhuckex75uy6mqmrdwnd4rg5gma" data-bluesky-embed-color-mode="dark"><p lang="en">Entry 31 - Migrating from the Bluesky PDS to the #BlackSky PDS with PDS MOOver #SharpieVLOG<br><br><a href="https://bsky.app/profile/did:plc:g7j6qok5us4hjqlwjxwrrkjm/post/3lw3hcuojck2u?ref_src=embed">[image or embed]</a></p>&mdash; dapurplesharpie 🔜 #ATProto_NYC (<a href="https://bsky.app/profile/did:plc:g7j6qok5us4hjqlwjxwrrkjm?ref_src=embed">@sharpiepls.com</a>) <a href="https://bsky.app/profile/did:plc:g7j6qok5us4hjqlwjxwrrkjm/post/3lw3hcuojck2u?ref_src=embed">Aug 10, 2025 at 4:39 PM</a></blockquote><script async src="https://embed.bsky.app/static/embed.js" charset="utf-8"></script></center>

You receive the final PLC code via email. If you are moving from an old self hosted PDS, double check that your SMTP is working properly on that server before attempting this or you won't receive the PLC.
# Future
* Docker [pdsadmin](https://github.com/cr0ssing/pdsadmin-docker)?
# More ATProto
* [BookHive](https://bookhive.buzz/) book tracking
* [Graze](https://www.graze.social/) custom feeds
* [recipe.exchange](https://recipe.exchange/) here is a recipe for [Cajun dirty rice](https://recipe.exchange/recipes/01JEY7M7SXDCKHKE3EC4Y4EJ1F)
* [Smoke Signal](https://smokesignal.events/) events & RSVPs
* [teal.fm](https://teal.fm/) music history ala last.fm