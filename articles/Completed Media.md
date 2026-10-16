


```dataview 
TABLE without id ("![|150](" + default(image,cover) + ")") AS Cover, default(title, "") as title
FROM "Media DB/games" OR "Media DB/movies" or "BookDB"
where played = True or watched = True or read = True


```

