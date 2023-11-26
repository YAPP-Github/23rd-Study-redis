## SearchAndQuery
- RediSearch 모듈은 Full Text Search 기능을 제공한다.

### Tag
- Use the @ symbol before the field name
- Surround the search term with curly braces

```
127.0.0.1:6379> FT.SEARCH books-idx "@isbn:{9781577312093}"
1) (integer) 1
2) "ru203:book:details:9781577312093"
3)  1) "title"
    2) "The Inner Reaches of Outer Space"
    3) "categories"
    4) "Religion"
    5) "published_year"
    6) "2002"
    7) "author_ids"
    8) "1572"
    9) "authors"
   10) "Joseph Campbell"
   11) "thumbnail"
   12) "http://books.google.com/books/content?id=5VOGAAAAQBAJ&printsec=frontcover&img=1&zoom=1&edge=curl&source=gbs_api"
   13) "subtitle"
   14) "Metaphor as Myth and as Religion"
   15) "isbn"
   16) "9781577312093"
   17) "average_rating"
   18) "4.22"
   19) "description"
```

```
127.0.0.1:6379> FT.SEARCH books-idx "@categories:{Fantasy}"
 1) (integer) 9
 ...
 
FT.SEARCH books-idx "@published_year:[2000 +inf]"
 1) (integer) 4349
 ...
 
FT.SEARCH users-idx "@last_login:[1607693100 +inf]"

FT.SEARCH books-idx "dogs|cats"

FT.SEARCH books-idx "@published_year:[2018 +inf]" SORTBY published_year DESC
```

### Stemming
```
127.0.0.1:6379> FT.SEARCH books-idx "@title:running"
1) (integer) 14
 2) "ru203:book:details:9780451197962"
 3)  1) "title"
     2) "The Running Man"
    
 ...
 
20) "ru203:book:details:9780671024185"
21)  1) "title"
     2) "Morgan's Run"
```
- Redis Search는 기본적으로 입력된 검색어의 "어근"(root form)을 찾아 유사한 단어들을 검색 결과에 포함시키는 '어근 환원'(Stemming) 기능을 사용한다.
- "run", "running", "runner" 등은 모두 같은 어근을 공유합니다. Redis Search는 이러한 어근을 기반으로 검색어와 유사한 단어들을 결과에 포함시켜, 사용자가 보다 광범위한 정보를 얻을 수 있다.


### Prefix Matching
```
127.0.0.1:6379> FT.SEARCH books-idx "atwood hand*"
```

### Aggregate
```
127.0.0.1:6379> FT.AGGREGATE books-idx * GROUPBY 0 REDUCE COUNT 0 AS total
1) (integer) 1
2) 1) "total"
   2) "6810"
   
127.0.0.1:6379> FT.SEARCH books-idx "@categories:{Fiction}" LIMIT 0 0
1) (integer) 2594
```
- LIMIT 절은 LIMIT {시작위치} {개수} 형식
- 0 0으로 설정된 경우, 전체 검색 결과의 개수를 확인할 수 있다.

```
127.0.0.1:6379> FT.AGGREGATE books-idx python GROUPBY 1 @categories
1) (integer) 5
2) 1) "categories"
   2) "Biography & Autobiography"
3) 1) "categories"
   2) "Fiction"
4) 1) "categories"
   2) "Performing Arts"
5) 1) "categories"
   2) "Humor"
6) 1) "categories"
   2) "Drama"
```
- books-idx 색인에서 python이 포함된 데이터를 찾아, categories 필드의 값에 따라 그룹화
