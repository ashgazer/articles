---
genre: classic
---
```dataview

TABLE WITHOUT ID key, length(rows) AS Count
FROM "BookDB"
WHERE contains(lower(join(Categories, ",")), lower(this.genre)) 
FLATTEN categories AS cat
GROUP BY cat




```




### Search by genre
```dataview
TABLE WITHOUT ID  author as Author, title as Title ,("![|100](" + cover + ")") AS Cover, this.genre as search_term
FROM "BookDB" 

WHERE contains(lower(join(Categories, ",")), lower(this.genre)) 



```

## short list

```dataview 
TABLE WITHOUT ID  author as Author, title as Title ,("![|100](" + cover + ")") AS Cover FROM "BookDB" 
where electronic = true
and read = false  
```



## readbooks

```dataview 
TABLE WITHOUT ID  author, title ,("![|100](" + cover + ")") AS Cover FROM "BookDB" 
where 1=1
and read = true  
```



![[BookDB.base]]
```