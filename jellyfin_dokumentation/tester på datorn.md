1. Fysisk kontroll (gör detta först, utan att koppla in disken till din vanliga dator)
Lyssna efter klick, malande eller tickande ljud när den startar → tyder på mekaniskt fel (läshuvud/motor). Om du hör detta: stäng av direkt och gå till punkt 5.
Kolla om disken syns i BIOS/UEFI vid uppstart överhuvudtaget.
2. Koppla in disken säkert på en annan dator
Använd en USB-till-SATA-adapter eller en extern docka, koppla inte in den som primär bootdisk.
Kolla i Windows Diskhantering / Linux lsblk om disken syns alls (även om den inte går att öppna).
3. SMART-status (avgör hur illa det är)
Linux: sudo smartctl -a /dev/sdX (kräver smartmontools)
Windows: CrystalDiskInfo
Titta efter: Reallocated_Sector_Count, Current_Pending_Sector, Uncorrectable_Error_Count. Höga värden = disken är fysiskt på väg att dö.
4. Filsystemtest (om disken syns men inte mountar)
Linux: sudo fsck -n /dev/sdX1 (bara -n, ingen skrivning, för att se skadan utan att riskera mer)
Windows: chkdsk X: /f men bara om du redan har säkrat vad du kan – chkdsk kan skriva och förstöra räddningsbara data.
5. Dataräddningsförsök (skrivskyddat, mot en image – inte disken direkt)
Gör först en sektor-image med ddrescue (inte vanlig dd) till en frisk disk – den hoppar över dåliga sektorer istället för att fastna:
ddrescue -d /dev/sdX rescue.img rescue.log
Kör sedan TestDisk/PhotoRec mot imagen, inte mot originaldisken. Så riskerar du inte att göra mer skada på originalet om något går fel.
Tolkning
Syns i BIOS + rimlig SMART + monterar men saknar filer → logiskt fel, TestDisk/PhotoRec löser troligen det.
Klickljud eller extremt höga SMART-fel → fysiskt fel, mjukvara hjälper inte, professionell räddning krävs om datan är viktig
