+++
# A section created with the Blank widget.
widget = "blank"  # See https://sourcethemes.com/academic/docs/page-builder/
headless = true  # This file represents a page section.
active = true  # Activate this widget? true/false
weight = 30  # Order that this section will appear.

# Note: a full width section format can be enabled by commenting out the `title` and `subtitle` with a `#`.
title = "Quickstart"
subtitle = ""

[design]
  # Choose how many columns the section has. Valid values: 1 or 2.
  columns = "2"

[design.background]
  # Apply a background color, gradient, or image.
  #   Uncomment (by removing `#`) an option to apply it.
  #   Choose a light or dark text color by setting `text_color_light`.
  #   Any HTML color name or Hex value is valid.

  # Background color.
  # color = "navy"
  
  # Background gradient.
  # gradient_start = "DeepSkyBlue"
  # gradient_end = "SkyBlue"
  
  # Background image.
  # image = "image.jpg"  # Name of image in `static/media/`.
  # image_darken = 0.6  # Darken the image? Range 0-1 where 0 is transparent and 1 is opaque.

  # Text color (true=light or false=dark).
  # text_color_light = true

[design.spacing]
  # Customize the section spacing. Order is top, right, bottom, left.
  # padding = ["0px", "0px", "0px", "0px"]
  padding = ["20px", "0", "20px", "0"]

[advanced]
 # Custom CSS. 
 css_style = ""
 
 # CSS class.
 css_class = ""
+++

Run Keyper docker image using docker cli:
```console
$ docker run  -p 8080:80 -p 8443:443 -p 2389:389 -p 2636:636 --hostname <hostname> --env FLASK_CONFIG=prod -it dbsentry/keyper
````
Run Keyper docker image using podman cli:
```console
$ podman run  -p 8080:80 -p 8443:443 -p 2389:389 -p 2636:636 --hostname <hostname> --env FLASK_CONFIG=prod -it docker.io/dbsentry/keyper
````
Either command starts a new container with keyper processes (openldap, gunicorn, nginx) running inside.

{{% alert note %}}
The above commands are for demo only and start a new container with LDAP server storing data within the container. That means on re-start, data would not persist. If you want data to persist, start the container with appropriate command line parameters. Refer to the documentation.
{{% /alert %}}

{{% alert note %}}
By default, this image creates an LDAP server for the company **Example Inc** and the domain **keyper.example.org**. By default, all passwords are set to *superdupersecret*. All the default settings can be changed by passing parameters to docker/podman command line or using a parameter config file.
{{% /alert %}}

**Customary 5 min installation Demo**

{{< youtube id="3332nHmKTYc" autoplay="true" >}}