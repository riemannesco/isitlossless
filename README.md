# isitlossless

A lot of FLAC files floating around aren't really lossless. Someone took an MP3 or an AAC, re-encoded it to FLAC, and now it has the size of a lossless file without the quality.

`isitlossless` looks at the audio itself to tell the difference. When it finds a fake, it tries to guess where it came from ("this was probably a 192 kbps MP3").

It's a small, cross-platform CLI written in Rust, including the FLAC decoder.

## Status

Work in progress, not usable yet. See [ROADMAP.md](ROADMAP.md) for where it's heading.

## License

Licensed under either of

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE))
- MIT license ([LICENSE-MIT](LICENSE-MIT))

at your option.
