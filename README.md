# Blog

### Local development setup

```sh
npm install hexo-cli -g
```
```sh
npm install
```
```sh
hexo server --draft
```
### Content management

##### Create page

```sh
hexo new page <path-name>
```
- Change value of `title:` and remove `date:` in `source/<path-name>/index.md`
- Add `<button-text>: /<path-name>` to `menu:` in `themes/landscape/_config.yml`

Restart development server

##### Create draft post
```sh
hexo new draft <title>
```

##### Publish post 

