# Postgres Pipeline: Especially Emphasizing Execution

These are the slides and my notes for a talk about the Postgres query pipeline,
in particular the executor phase.

It was given at PGCon 2023 in Ottawa on 1 June 2023.
It is built on [reveal.js](https://github.com/hakimel/reveal.js/).

You also can get [all the slides as a PDF](slides.pdf),
or [read the slides' Markdown with speaker notes](slides.md).

The video is [on my site](https://illuminatedcomputing.com/pages/pgcon2023/).

## Development

You can run the slides locally by saying:

```
npm install
npm start
```

You can get a slideshow-able PDF by going to http://localhost:8001/?print-pdf *in Chrome* and printing to a PDF. This is what you want to save as `slides.pdf`.

You can include speaker notes by going to http://localhost:8001/?print-pdf&showNotes=separate-page *in Chrome*, printing to a PDF, then printing the PDF. It uses a custom hack to get separate-page speaker notes with a white background. This is what you want to print before you give the talk.
