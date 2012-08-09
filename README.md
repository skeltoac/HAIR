# HAIR
## HTTP Accept Image Ranges

A protocol allowing web browsers to request optimal representations of images according to their 

* physical display properties,
* network connection properties,
* magnification level,
* visible extents,
* user preferences,
* etc.,

with existing HTML, CSS, and JS.

### Problem:

Displays with high pixel ratios look their best with higher-resolution graphical elements. So do 1:1 devices when magnifying documents. Device vendors spent a great deal more time on integrating hi-res graphics in their operating systems than they invested in web graphics. New methods are appearing every day involving changes to HTML, CSS, JS, and even cookies.

All of the existing solutions for web graphics are predicated on holding browser and server technologies constant and varying the applications that run on top of them (HTML, CSS, JS). My own work in this field (WordPress.com's devicepx.js) has taught me that this entire approach should be made redundant as soon as possible.

A step in the right direction would be to move the problem down the stack, out of the application and into the underlying layer: HTTP clients and servers. Web pages and their creators and maintainers vastly outnumber the software options for browsers and servers. Holding HTML, CSS and JS constant and varying client and server software will be vastly more effective and efficient.

### Solution:

Transfer metadata in requests and responses using existing HTTP/1.1 header specifications. Request headers specify the target rendering dimensions or the device pixel ratio; responses tailored to these specifications include headers which indicate the variance and the maximum dimensions of the underlying resource. This is a minimalistic form of content negotiation. I will refer to this technique in general by an acronym from the title: HAIR.

The Accept header is my first choice for a HAIR request header:

* The field already deals with media types.
* The field may include extensible parameters.
* Precedence and fallback are built in.

The Content-Type header can also carry extensible parameters and is therefore my first choice for response metadata when it confirms a match in the Accept header. The Pragma header can be used to transmit any other HAIR metadata in either direction.

The kind of metadata that belongs in the Accept header is anything that can help the server find the optimal variant of a resource on the very first request for that resource.

### Example 1: img with dimensions

The simplest case is rendering an image with dimensions provided in HTML attributes. Given a device with a pixel ratio of 2, a 100px square image, and no reason such as bandwidth cost or speed to throttle image quality, the browser should ask for a 200x200 representation. This is a basic HAIR transaction:

```
HTML:
    <img src="/img/bob" width="100" height="100" … />

HAIR Request:
    GET /img/bob HTTP/1.1
    …
    Accept: image/*; wpx=200; hpx=200, image/*

HAIR Response:
    HTTP/1.1 200 OK
    …
    Content-Type: image/png; wpx=200; hpx=200
    Pragma: owpx=4000, ohpx=4000
    Accept-Ranges: crop
    
    [200px square image]
```

Absent the HAIRy Accept header, the server would have responded as any other server with a 100px image and a parameterless Content-Type header. Seeing the HAIRy Accept header, the server has located or generated a 200px image and indicated that the response varies from the default by echoing the parameters from the request.

The optional Pragma response header includes the dimensions of the original. In this case, the server indicates that it has a 4000px source image. The source image dimensions could be used by clients with variable pixel ratios (browsers allowing visual magnification or zooming) to decide whether to request a larger variant when the user increases the magnification factor.

The Accept-Ranges response header announces that the server accepts Range headers with "crop" units. Cropping might be used to load only the visible extents of a very large image. This advanced feature is mentioned here as an aside; it is not expected to be included in early implementations.

### Example 2: img without dimensions

Often the browser can not compute the rendered size of an image without first loading the image to learn its natural size. In this case, a HAIRy client can still send information to help the server select an optimal representation. Instead of pixel dimensions, the browser transmits its device pixel ratio (`dpr=2`) and the server responds optimally:

```
HTML:
    <img src="/img/bob" … />

HAIR Request:
    GET /img/bob HTTP/1.1
    …
    Accept: image/*; dpr=2, image/*

HAIR Response:
    HTTP/1.1 200 OK
    …
    Content-Type: image/png; dpr=2
    Pragma: dwpx=100, dhpx=100
    
    [200px square image]
```

The server contains a default representation for `/img/bob` which is a 100px square image. It computes the optimum size by multiplying by 2 (the dpr), arriving at a 200px square image. The Pragma header includes the default representation's dimensions because the client needs this information to correctly flow the page. It would be an error for the client to assume that dividing the response's dimensions by 2 (`dpr=2`) would result in the default dimensions, since it is permissible to send a 150px image if that is the best available representation.

### Compatibility and Caching

HAIR requests are backwards-compatible with HAIRless servers since they specify a fallback type of `image/*` (or whatever else the client prefers). HAIRy clients revert to pre-HAIR behavior when responses lack the headers that indicate server-side HAIR processing. Similarly, HAIRy servers maintain compatibility with HAIRless clients since HAIRy server behavior is only triggered by HAIRy request headers.

Besides being the most efficient way to improve graphics on the web, HAIR must also be cache-friendly. It is likely that advanced recommendations will refer to Etag, If-None-Match, and Vary headers.

### Caveats

I have intentionally glossed over the implementation details outside of the HAIR protocol. On the client side, my knowledge of HTML rendering algorithms is just enough to know that this proposal represents a significant undertaking by browser makers.

The new logic on the server side will be even more complex since servers will be asked to choose or generate an optimal representation of an image. One implementation might work by storing the original image and a file containing metadata such as the default reduction size. Another might develop a new type of file archive which contains several variants and parameters for choosing between them.

### Art Direction

A designer might decide that an image should be cropped differently for low-resolution variants to preserve important details. How designers will transmit these specifications to servers will depend on how server makers design their new image metadata or archive files.

### Archive Formats

Of the many possible archive formats, a filesystem directory may provide the easiest path to implementation in many servers. It is also portable among server platforms as well as desktops where archives will be generated by designers and the tools that will be developed for the purpose. The directory must contain at least one image: the default. Everything else is optional.

In the following listing, the first two files would be sufficient to make /img/bob a fully HAIRy resource. The remaining files may be provided by the author or generated and cached by the server.

```
/img/bob/index.jpeg
/img/bob/original.png
/img/bob/metadata.json
/img/bob/source-200x200.png
/img/bob/dpr=1.5.jpeg
/img/bob/dpr=2.jpeg
/img/bob/pxw=200.jpeg
```

The `index.*` file pattern is well-established. In this case, the designer has chosen jpeg as the optimal format. The server can rapidly respond to HAIRless requests with the contents of the index file; the Content-Type and Content-Disposion headers can be composed from the file's name without sniffing its contents.

The server can use the `original.*` file as the source when generating other representations as requested. Its dimensions can be provided or cached in a metadata file to save processing time.

Additional source images may be provided for generating smaller sizes. The `source-WxH.*` pattern specifies sources for generating variants up to WxH. Designers may also provide specific versions of an image for display at lower or higher pixel ratios if patterns like `dpr=2.*` are supported.

The `metadata.json` file may also include processing instructions such as resampling and filtering algorithms best suited for the given image. For example, a designer might decide that a photograph may be downscaled with bicubic resampling and then filtered with unsharp-mask, a line drawing may be downscaled but not filtered, pixel art should be upscaled by nearest-neighbor, etc.

### File extensions

The omission of file extensions from HTML and HTTP requests is purposeful. It simplifies the HTML while allowing the selection of the optimal format by the server or designer.

Format selection criteria may vary with request parameters and design parameters. A small representation may be optimized for Content-Length while sizes closer to the original may sacrifice compression for fidelity. Image URIs need not suggest the content type.

### Other client factors

As briefly mentioned before the first HAIR transaction example, clients may also use information about connection speed, bandwidth cost, user settings or other external factors in their internal HAIR algorithms. A high-res mobile device operating on a slow or expensive network could use HAIR to request reduced-quality images. The same behavior could be triggered by switching to battery power or reaching a low-battery threshold.

### Future

While I have no illusion that adding HAIR support to browsers and servers would be a trivial effort, the basic logic is simple: if the render dimensions are known to the client without reference to the resource then include them in the request, otherwise include the target pixel ratio; if the server can locate or generate a representation better suited than the default for that resource, serve it and echo the parameters.

HAIR's final form will differ from the examples in the initial version. The specific headers and parameters is not the point; it is expected that better alternatives will be found with early experimentation and that implementations will converge on a workable standard.

This proposal is admittedly far from mature. It is a start.