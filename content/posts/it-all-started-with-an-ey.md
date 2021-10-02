+++
author = "Hannes Knutsson"
title = "It All Started With an Ey"
date = 2021-10-02T17:01:26+02:00
tags = [
  "token"
  "JSON"
  "Base64"
]
draft = false
+++

# ey

Is the start of all bearer tokens when accessing APIs and similar resources. I have not really been thinking about it much until I realized that tokens are Base64 encoded JSON objects.

JSON objects allways start with an ```{``` to encapsulate the contents of the object, so it makes sense that all tokens start with the same.

Just thought that was a fun fact when I thought of it, so why not put it out there.
