# step 1 logga in till Radarr

```
http://100.109.27.41:7878
```

# kontrollera Hardlinks påslaget

1. gå till "settings"
2. gå till "Media Managment"
3. scrolla ner till "Impoorting"
4. hitta "Use Hardlinks instead of Copy"
5. klicka på checkrutan så att den är i kryssad


# kontrollera Remove Completed påslaget

1. gå till "settings"
2. gå till "Download Clients"
3. klicka på "qBittorent"
4. scrolla längst ned till Completed Download Handling
5. kryssa på "Remove Completed"


# Bekräfta att downloads och movies ligger på samma fysiska disk
### downloads/ och jellyfin/movies/ måste ligga på samma disk (Spomini) för att hardlinks ska fungera. Om de någonsin flyttas till olika diskar, faller hela systemet tillbaka på 'copy' istället, vilket dubblerar diskanvändningen.

kör koden 
```
df -h /media/miska/Spomini/downloads /media/miska/Spomini/jellyfin/movies
```

du ska få output 


```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb1       1.8T  326G  1.4T  19% /media/miska/Spomini
/dev/sdb1       1.8T  326G  1.4T  19% /media/miska/Spomini
```
