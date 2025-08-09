+++
author = "hannibal"
categories = ["golang", "crd-to-sample-yaml", "cty"]
date = "2025-08-09T01:01:00+01:00"
title = "CTY now supports using a custom CSS for the HTML output"
url = "/2025/08/09/crd-to-sample-yaml-with-custom-css"
comments = true
+++

# CTY now supports using a custom CSS for the HTML output

Hey folks.

My tool, cty[^1] now supports adding custom CSS to the HTML output. In an effort to prevent cross-site scripting
and other CSS nasties the CSS is pretty limited, sanitized and escaped.

Using `--css-file` now, it's possible to give it a CSS file that will overwrite everything inside the main HTML file.
This allows you to add any customization that you would like to apply to the end result during generation. No more
post processing, post editing necessary.

You can read more about it here[^2].

Enjoy!
Thanks for reading,
Gergely.


[^1]: https://github.com/Skarlso/crd-to-sample-yaml
[^2]: https://github.com/Skarlso/crd-to-sample-yaml?tab=readme-ov-file#custom-css