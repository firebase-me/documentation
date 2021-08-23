
Do your images take a long time to load from [Cloud Storage for Firebase](https://firebase.google.com/products/storage)? That's probably because Firebase Storage is not a  [**C**ontent **D**elivery **N**etwork](https://en.wikipedia.org/wiki/Content_delivery_network). The loads times maybe even higher if your storage bucket is located far away from you. 

This article is a step-by-step guide to set up [Bunny CDN](https://bunny.net) with Cloud Storage for Firebase. 

#### 1. Create a Pull Zone:
A pull zone essentially tells Bunny CDN's system where to find your files and how to serve them to your users. Bunny CDN offers a hostname (a domain over which you can access your images) for a pull zone but you also have an option to set a custom domain name. Once you create an account, navigate to Pull Zones and click "Add Pull Zone":


![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1629644298078/YsuC3NUZH-.png)

Enter your preferred hostname from where your images will be served and origin URL from where Bunny CDN will fetch the images and cache on it's edge server.


![image2.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1629651617749/pb_zUnC0nd.png)

This means if you request a resource at `https://new-pullzone.b-cdn.net/images/fire.png`, Bunny CDN will fetch it from `https://firebasestorage.googleapis.com/v0/b/[PROJECT_ID]/o/images/fire.png` once and cache it on it's edge servers. 

#### 2. Configure cache expiration time
By default, Bunny CDN respects `Cache-Control` header set by the origin which in case of Firebase storage is `cache-control: private, max-age=0` unless you've changed it using [metadata](https://firebase.google.com/docs/reference/js/firebase.storage.SettableMetadata#cachecontrol). That means your files won't be cached and Bunny CDN will just be a proxy server. You must override the `Cache-Control` header from origin: 

![image3.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1629652195441/q3X-_i9Eh.png)

#### 3. Accessing the images:
All the necessary configurations are done. To access the images via CDN, you must use the hostname provided by Bunny CDN instead of the original download URLs from Firebase. You just need to replace the origin URL (that we had configured in Bunny CDN) with the hostname:

```js
const originUrl = "https://firebasestorage.googleapis.com/v0/b/[PROJECT_ID]/o"
const hostname = "https://new-pullzone.b-cdn.net"
const storage = firebase.storage();
const imageRef = storage.ref("images/fire.png");

imageRef.getDownloadURL().then((url) => {
    const cdnUrl = url.replace(originUrl, hostname);
    console.log(`CDN URL: ${cdnUrl}`)
    // render the image in your application
})
```
To confirm that your images are being served from Bunny CDN cache, you can check for `cdn-cache` header in the response. If it's value is `HIT`, that means the image was server from cache. 

Do note that the URLs must have the `?token=` query parameter for the first time else Firebase will respond with a 403 error. 

Congrats! Your images now load faster than before. 

However, there's a drawback. Once the image is cached on edge, anyone will be able to access it without the `?token=` parameter which defeats the purpose of having it and the security rules. Even though security rules just prevent random users from fetching the download URLs and not accessing the images if authorized users shared it with others, this may not be ideal in some cases. 

One way around would be to use those tokens ([UUID](https://www.npmjs.com/package/uuid)) as the image name (e.g. `img_9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d.png`) but you cannot revoke these unless you reupload the image (to rename the image with new token). Bunny CDN also offers [token authentication](https://support.bunny.net/hc/en-us/articles/360016055099) which are technically signed URLs (but with CDN) if you really need to keep images private. 

#firebase 

