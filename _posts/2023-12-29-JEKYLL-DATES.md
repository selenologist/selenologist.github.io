---
title: Jekyll Date Strings
date: 2023-12-29 14:21:51+11:00
categories: [time, linux]
tags: [time, jekyll, bash]
---

For my first serious post, why not one I'll probably constantly refer to myself every time I need to make a post?

Jekyll post filenames use the same format as the result of the command `date -I`. Date strings (at least as used with Chirpy) have the same format as `date --rfc-3339='seconds'`.

So I made this post with the simple one-liner, `date --rfc-3339='seconds' > _posts/$(date -I)-JEKYLL-DATES.md`. Then I just have to fill in the Front Matter around the date string and away we go. As I write this, I feel I can probably write a better script which templates that too. So, here's that bash script:
```bash
#!/bin/bash

# Get the title from the first argument
title="$1"

# Generate a slug from the title, replacing spaces with hyphens
slug="${title// /-}"

# Create the filename with the date and slug
filename="_posts/$(date -I)-${slug}.md"

# Write the front matter, including the title and date
cat << EOF > "$filename"
---
title: $title
date: $(date --rfc-3339='seconds')
categories: []
tags: []
---

EOF

echo "Jekyll post created at: $filename"
```
