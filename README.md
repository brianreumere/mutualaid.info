# mutualaid.info

All configuration and site assets for [https://mutualaid.info](mutualaid.info).

## Ansible

The `hugo` role configures an OpenBSD server to serve a static site from `/var/www/htdocs/hugo` with `httpd`. The `www.yml` playbook executes the role.

```
ansible -m raw -a "doas pkg_add python3" -i hosts www
ansible-playbook -i hosts www.yml
```

## Hugo

This is the actual website content. It can be built by running the `hugo` command from the `hugo/mutualaid.info` directory and then copied to a web server.

```
cd hugo/mutualaid.info
hugo
scp -r public/* mutualaid.info:/var/www/htdocs/hugo
```

### Webmentions

It's TBD how I'll support webmentions (I'd like to self-host something). These are some options:

* go-jamming: [blog post](https://brainbaking.com/post/2021/05/beyond-webmention-io/) and [repo](https://github.com/wgroeneveld/go-jamming)
* webmentiond: [blog post](https://zerokspot.com/weblog/2020/06/14/setting-up-webmentiond/) and [repo](https://github.com/zerok/webmentiond)

TBD how to render them. I would like to avoid Javascript but rebuilding the Hugo site on a schedule seems gross too...

I'm also wary of privacy and consent concerns. See [The Indieweb privacy challenge (Webmentions, silo backfeeds, and the GDPR)](https://sebastiangreger.net/2018/05/indieweb-privacy-challenge-webmentions-backfeeds-gdpr/) and [this Twitter thread that has stuck with me](https://twitter.com/redlightvoices/status/1021747354398076928).

### IndieAuth

I'm not sure if I need this or would use it, so it's lower priority. Or maybe it's needed for Webmentions?