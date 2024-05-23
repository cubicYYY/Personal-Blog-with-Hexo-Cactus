A solution for conflicts between LaTex and Hexo regex matching engine:
In the file of node_modules/marked/lib/marked.js:

```
link: /^!?\[(inside)\]\(href\)/,
reflink: /^!?\[(inside)\]\s*\[([^\]]*)\]/,
nolink: /^!?\[((?:\[[^\]]*\]|[^\[\]])*)\]/,
strong: /^__([\s\S]+?)__(?!_)|^\*\*([\s\S]+?)\*\*(?!\*)/,
em: /^\b_((?:[^_]|__)+?)_\b|^\*((?:\*\*|[\s\S])+?)\*(?!\*)/,
code: /^(`+)([\s\S]*?[^`])\1(?!`)/,
br: /^ {2,}\n(?!\s*$)/,
del: noop,
text: /^[\s\S]+?(?=[\\<!\[_*`]| {2,}\n|$)/
```

Change the line

```em: /^\b_((?:[^_]|__)+?)_\b|^\*((?:\*\*|[\s\S])+?)\*(?!\*)/,```

to

```em:  /^\*((?:\*\*|[\s\S])+?)\*(?!\*)/,```

This will disable the effort of trying to match _ for the regular expression engine.