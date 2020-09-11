+++
# A section created with the Blank widget.
widget = "blank"  # See https://sourcethemes.com/academic/docs/page-builder/
headless = true  # This file represents a page section.
active = true  # Activate this widget? true/false
weight = 20  # Order that this section will appear.

# Note: a full width section format can be enabled by commenting out the `title` and `subtitle` with a `#`.
title = "KEYPER"
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

Keyper is an SSH Public Key Manager. It standardizes and centralizes the storage of SSH public keys for all Linux users in your organization. Saving significant time and effort it takes to manage SSH public keys on each Linux Server. Keyper is a container occupying less than 100MB. It is launched either using Docker or Podman. You can be up and running within minutes instead of days.

Features include:
- Public key storage
- Public Key Expiration
- Forced Key rotation
- Streamlined provision or deprovision of users
- Segmentation of Servers using groups
- Define policies to restrict user's access to server(s)
- Centralized user account lockout
- Docker container

{{< figure library="true" src="keyper.png"  >}}