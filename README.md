> ### ⚠️ This repository is archived
>
> This is the **original UndergroundBB implementation** (Node.js + Express + React + PostgreSQL +
> Redis). It is preserved here for reference and is no longer developed.
>
> Development has moved to a ground-up rewrite at
> **[pwntato/undergroundbb](https://github.com/pwntato/undergroundbb)** — Go on AWS Lambda with a
> single DynamoDB table, and crypto moved fully client-side so the server only ever stores
> ciphertext and public keys.

# UndergroundBB

UndergroundBB is a place where people can communicate with each other and know their conversations are safe. All post titles and bodies are fully encrypted such that it requires one of the group's user's password, which aren't stored, even in hash form, on the server end. New users can only be added to a group by users already in the group with a role of Ambassador or Admin, thus providing a chain of trust. The weakest link in the chain is user passwords, so it is recommended to use strong passwords.

# Setup the development environment

1. Clone the repo: `git clone git@github.com:pwntato/undergroundbb.git`
1. Change into the project directory: `cd undergroundbb`
1. Create `.env` file
   1. Copy the sample file: `cp .env.sample .env`
   1. Fill out the missing values in`.env`, anything alphanumeric should be fine for development, but choose strong secrets for production deploys
1. Clean out any previous installs: `docker-compose down -v`
1. Create the docker containers: `docker-compose build --no-cache`
1. Start the docker containers: `docker-compose up`
1. Make sure the smoke tests pass
   1. Go to the frontend folder: `cd frontend`
   1. Run the Playwright tests: `npm run test:e2e`

# How it works

## The chain of trust

Anyone can create one or more groups and they will have the role of Admin. That is the highest role. It can invite more users to the group, change the roles of users in the group, and various other group settings.

Users invited to a group are invited as Members. They are able to create and view posts and comments in the group. All users must be invited by a user with a role of Admin or Ambassador in the group.

If an Admin user promotes a Member to Ambassador, that means the user is trusted enough to be able to invite new Member users.

This means that the only way into the group is getting an invitation from a trusted member. This is the chain of trust.

## The encryption

UndergroundBB uses AES-256 for symmetric encryption and RSA-3072 for asymmetric encryption.

### Initial signup

A new user is created, which generates a new RSA-3072 key pair. The private key is AES-256 encrypted using a salted hash of the user's password. The RSA public key, the AES encrypted RSA private key, and the salt used for the hash (not the hash itself) are stored on the user in the database.

### Login

The user submits their username and password. They are looked up from the database by username. Then the salt stored on their user record is used to hash their password. The resulting hash is used to decrypt the AES encrypted RSA private key. A test message is encrypted with the user's RSA public key and then decrypted with the user's RSA private key. If the decrypted message matches the initial message, the user is considered logged in.

Once the user is logged in, a random AES key is generated. This key is used to encrypt the user's RSA private key and store it in the user's session (stored in Redis). The random AES key is then stored in a cookie that is sent to the user's browser. Both the session and the cookie have a fairly short expiration.

This means someone would need both the client side token and the session data to decrypt the RSA private key from the session.

### Create a group

When a user creates a group, a new random AES key is generated and encrypted to the user's RSA public key. This key is stored in the membership table along with the user's role in the group, which is Admin for all groups they create.

### Invite a user

Any user with a role of Admin or Ambassador can invite users. The user must exist before they are invited to a group. 

In order to invite a new user, an existing user needs to use their RSA private key to decrypt the AES group key that was encrypted to their RSA public key and stored on their membership record. Then they encrypt the AES group key to the new user's RSA public key. The result is stored on a new membership record for the invited user along with the role they have in the group, which is Member by default.

### Posts and comments

When interacting with posts and comments for a group, the user posts their random AES session token with every call to the server. This is used to decrypt their RSA private key from the Redis session. Then their RSA private key is used to decrypt the group's AES key that is stored on the user's membership record. The AES group key is used to encrypt and decrypt all posts, post titles, and comments within the group.
