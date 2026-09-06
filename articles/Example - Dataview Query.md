## short list

```dataview 
TABLE WITHOUT ID  author, title ,("![|100](" + cover + ")") AS Cover FROM "BookDB" 
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