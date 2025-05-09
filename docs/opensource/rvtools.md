# RVTools Report Script

Bu script, birden fazla VMware vCenter ortamına bağlanır, RVTools ile vm inventory bilgilerini export alır
ve alınan Excel dosyalarını tek bir özet raporda birleştirir. Daha sonra belirtilen adrese alınan exportları mail olarak gönderir.

> RVTools gibi envanter datası toplayan araçlarla vCenter'a erişim sağlanırken "Read-Only" yetkili bir kullanıcı hesabı kullanılması kesinlikle tavsiye edilir.

> RVTools, varsayılan olarak .ini dosyasında kullanıcı adı ve şifre bilgilerini düz metin olarak saklar. Bu durum, özellikle paylaşımlı sistemlerde veya otomatik çalışan script’lerde güvenlik açığı oluşturabilir. Dolayısı ile RVToolsPasswordEncryption kullanınız.

> Sadece iki VMware ortam için script kısaltıldı. Ne kadar ortamınız varsa script içerisinde kodu değiştirip kullanabilirsiniz.

Windows "Schedule Task" veya Linux ortamlarda Cron oluşturup otomatize edebilirsiniz.  
Faydalı olması dileğiyle.

# Files
```bash
git clone https://github.com/emrahuludag/rvtools_inventory.git
```

```bash
 cd C:\Program Files (x86)\Robware\RVTools\ 
 ./RVToolsPasswordEncryption.exe yourPassword
```

```bash
# =============================================================================================================
# Script:    RVTools_Report.ps1
# Date:      July, 2024
# By:        Emrah ULUDAG
# =============================================================================================================
<#
.SYNOPSIS
This script connects to multiple VMware vCenter environments, collects virtual machine inventory data using RVTools,
and exports the data to Excel files. The resulting files are then optionally merged and summarized using pandas,
filtered by selected columns, and saved to a structured output directory. Final reports can be sent via email
to designated recipients.
#>

[string] $RVToolsPath = "C:\Program Files (x86)\Robware\RVTools"

set-location $RVToolsPath

# =====================================
# RVTools - A-site-vmwarename
# =====================================
[string] $VCServer = "x.x.x.x"                    
[string] $User = "username@vsphere.local"                                                    
[string] $EncryptedPassword = "_RVToolsV2encripytedpassword"
[string] $XlsxDir1 = "C:\RVTools"
[string] $XlsxFile1 = "A-site-vmwarename.xlsx"
[string] $XlsxFileoutput = "C:\RVTools\A-site-vmwarename.xlsx"


# Start cli of RVTools for A-site-vmwarename
Write-Host "Start export for vCenter $VCServer" -ForegroundColor DarkYellow
$Arguments = "-u $User -p $EncryptedPassword -s $VCServer -c ExportvInfo2xlsx -d $XlsxDir1 -f $XlsxFile1 -DBColumnNames -ExcludeCustomAnnotations"

Write-Host $Arguments
$Process = Start-Process -FilePath "C:\Program Files (x86)\Robware\RVTools\RVTools.exe" -ArgumentList $Arguments -NoNewWindow -Wait -PassThru

if($Process.ExitCode -eq -1)
{
    Write-Host "Error: Export failed! RVTools returned exitcode -1, probably a connection error! Script is stopped" -ForegroundColor Red
    exit 1
}
$OutputFile = $XlsxFileoutput


# =====================================
# RVTools - B-site-vmwarename
# =====================================

[string] $VCServer = "x.x.x.x"                    
[string] $User = "username@vsphere.local"                                                    
[string] $EncryptedPassword = "_RVToolsV2encripytedpassword"
[string] $XlsxDir1 = "C:\RVTools"
[string] $XlsxFile1 = "B-site-vmwarename.xlsx"
[string] $XlsxFileoutput = "C:\RVTools\B-site-vmwarename.xlsx"

# Start cli of RVTools for B-site-vmwarename
Write-Host "Start export for vCenter $VCServer" -ForegroundColor DarkYellow
$Arguments = "-u $User -p $EncryptedPassword -s $VCServer -c ExportvInfo2xlsx -d $XlsxDir1 -f $XlsxFile1 -DBColumnNames -ExcludeCustomAnnotations"

Write-Host $Arguments

$Process = Start-Process -FilePath ".\RVTools.exe" -ArgumentList $Arguments -NoNewWindow -Wait -PassThru

if($Process.ExitCode -eq -1)
{
    Write-Host "Error: Export failed! RVTools returned exitcode -1, probably a connection error! Script is stopped" -ForegroundColor Red
    exit 1
}

$OutputFile = $XlsxFileoutput

# =====================================
# RVTools Send Mail
# =====================================

set path=%path%;"C:\Program Files (x86)\Robware\RVTools"

[string] $SMTPserver="relay-mail-server-ip"
[string] $SMTPport="25"
[string] $Mailto="emrahuludag@gmail.com"
[string] $Mailto="group-mail@domain.local"
[string] $Mailfrom="rvtools.report@domain.local"
[string] $Mailsubject="RVTools Inventory Report"
[string] $AttachmentDir="C:\rvtools\"
[string] $XlsxDir = "C:\RVTools\"
[string] $XlsxFile1 = "A-site-vmwarename.xlsx"
[string] $XlsxFile2 = "B-site-vmwarename.xlsx"

# =====================================
# Start RVTools Merge All files
# =====================================

$inputFiles = "$XlsxDir\$XlsxFile1;$XlsxDir\$XlsxFile2"

.\RVToolsMergeExcelFiles.exe -input "$inputFiles" -output "C:\rvtools\Rvtools_merged.xlsx" -overwrite -verbose

# ====================================
# Sending Mail
# ====================================
.\rvtoolssendmail.exe /smtpserver $SMTPserver /smtpport $SMTPport /mailto $Mailto /mailfrom $Mailfrom /mailsubject "RVTools Inventory Report Merged" /attachment C:\rvtools\Rvtools_merged.xlsx
.\rvtoolssendmail.exe /smtpserver $SMTPserver /smtpport $SMTPport /mailto $Mailto /mailfrom $Mailfrom /mailsubject "RVTools Inventory A-site-vmwarename.xlsx Report " /attachment $XlsxDir\$XlsxFile1
.\rvtoolssendmail.exe /smtpserver $SMTPserver /smtpport $SMTPport /mailto $Mailto /mailfrom $Mailfrom /mailsubject "RVTools Inventory B-site-vmwarename.xlsx Report " /attachment $XlsxDir\$XlsxFile2

```

# OUTPUT

```bash
PS C:\Program Files (x86)\Robware\RVTools> C:\Users\euludag\Desktop\RVTools_script\rvtools_export_script.ps1
Start export for vCenter A-site-vmwarename
-u username@vsphere.local -p XXXX -s 10.2.A-site-vmwarename.xlsx.100 -c ExportvInfo2xlsx -d C:\RVTools -f A-site-vmwarename.xlsx -DBColumnNames -ExcludeCustomAnnotations
Start export for vCenter B-site-vmwarename
-u username@vsphere.local -p XXXXX -s B-site-vmwarename.xlsx -c ExportvInfo2xlsx -d C:\RVTools -f B-site-vmwarename.xlsx -DBColumnNames -ExcludeCustomAnnotations

C:\Program Files (x86)\Robware\RVTools
Check input C:\RVTools\\A-site-vmwarename.xlsx
Ok
Check input C:\RVTools\\B-site-vmwarename.xlsx
Ok
05:05:17 Copy C:\RVTools\\kzalavmmgmt.xlsx to C:\rvtools\Rvtools_merged.xlsx
05:05:17 Open XLWorkbook C:\rvtools\Rvtools_merged.xlsx
05:05:18 Open XLWorkbook C:\RVTools\\A-site-vmwarename.xlsx
05:05:18 Merge XLWorkbooks
05:05:18 Processing worksheet vInfo from C:\RVTools\\B-site-vmwarename.xlsx
05:05:18 Processing worksheet vMetaData from C:\RVTools\\B-site-vmwarename.xlsx
05:05:18 Dispose XLWorkbook C:\RVTools\\B-site-vmwarename.xlsx

Mail send to: emrahuludag@gmail.com
RVToolsSendMail: Terminated normally.
Mail send to: emrahuludag@gmail.com
RVToolsSendMail: Terminated normally.

PS C:\Program Files (x86)\Robware\RVTools> 
```


Tüm envanteri topladık ama istediğimiz alanlar olsun sadece isterseniz;
RVTools’tan alınmış birden fazla Excel dosyasını aynı klasöre atıp bu script’i çalıştırıyorsunuz.

Script, sadece belirttiğiniz alanları filtreleyerek tüm verileri birleştiriyor ve size temiz, sade ve odaklanmış bir dosya sunuyor.

Böylece envanterden “işe yarayan” veriye hızlıca ulaşabilirsiniz.


#

```bash
# RVTools Report Script

Bu script, birden fazla VMware vCenter ortamına bağlanır, RVTools ile envanter verilerini dışa aktarır
ve alınan Excel dosyalarını tek bir özet raporda birleştirir. Daha sonra belirtilen adrese mail olarak gönderir.

> Not: RVTools gibi envanter datası toplayan araçlarla vCenter'a erişim sağlanırken "Read-Only" yetkili bir kullanıcı hesabı kullanılması kesinlikle tavsiye edilir.
> RVTools, varsayılan olarak .ini dosyasında kullanıcı adı ve şifre bilgilerini düz metin olarak saklar. Bu durum, özellikle paylaşımlı sistemlerde veya otomatik çalışan script’lerde güvenlik açığı oluşturabilir. Dolayısı ile RVToolsPasswordEncryption kullanınız.
> Not: Sadece iki VMware ortam için script kısaltıldı. Ne kadar ortamınız varsa script içerisinde kodu değiştirip kullanabilirsiniz.

Windows "Schedule Task" veya Linux ortamlarda Cron oluşturup otomatize edebilirsiniz.  
Faydalı olması dileğiyle.

```bash
 cd C:\Program Files (x86)\Robware\RVTools\ 
 ./RVToolsPasswordEncryption.exe yourPassword
```

```bash
# =============================================================================================================
# Script:    RVTools_Report.ps1
# Date:      July, 2024
# By:        Emrah ULUDAG
# =============================================================================================================
<#
.SYNOPSIS
This script connects to multiple VMware vCenter environments, collects virtual machine inventory data using RVTools,
and exports the data to Excel files. The resulting files are then optionally merged and summarized using pandas,
filtered by selected columns, and saved to a structured output directory. Final reports can be sent via email
to designated recipients.
#>

[string] $RVToolsPath = "C:\Program Files (x86)\Robware\RVTools"

set-location $RVToolsPath

# =====================================
# RVTools TAV - A-site-vmwarename
# =====================================
[string] $VCServer = "x.x.x.x"                    
[string] $User = "username@vsphere.local"                                                    
[string] $EncryptedPassword = "_RVToolsV2encripytedpassword"
[string] $XlsxDir1 = "C:\RVTools"
[string] $XlsxFile1 = "A-site-vmwarename.xlsx"
[string] $XlsxFileoutput = "C:\RVTools\A-site-vmwarename.xlsx"


# Start cli of RVTools for A-site-vmwarename
Write-Host "Start export for vCenter $VCServer" -ForegroundColor DarkYellow
$Arguments = "-u $User -p $EncryptedPassword -s $VCServer -c ExportvInfo2xlsx -d $XlsxDir1 -f $XlsxFile1 -DBColumnNames -ExcludeCustomAnnotations"

Write-Host $Arguments
$Process = Start-Process -FilePath "C:\Program Files (x86)\Robware\RVTools\RVTools.exe" -ArgumentList $Arguments -NoNewWindow -Wait -PassThru

if($Process.ExitCode -eq -1)
{
    Write-Host "Error: Export failed! RVTools returned exitcode -1, probably a connection error! Script is stopped" -ForegroundColor Red
    exit 1
}
$OutputFile = $XlsxFileoutput


# =====================================
# RVTools TAV - B-site-vmwarename
# =====================================

[string] $VCServer = "x.x.x.x"                    
[string] $User = "username@vsphere.local"                                                    
[string] $EncryptedPassword = "_RVToolsV2encripytedpassword"
[string] $XlsxDir1 = "C:\RVTools"
[string] $XlsxFile1 = "B-site-vmwarename.xlsx"
[string] $XlsxFileoutput = "C:\RVTools\B-site-vmwarename.xlsx"

# Start cli of RVTools for B-site-vmwarename
Write-Host "Start export for vCenter $VCServer" -ForegroundColor DarkYellow
$Arguments = "-u $User -p $EncryptedPassword -s $VCServer -c ExportvInfo2xlsx -d $XlsxDir1 -f $XlsxFile1 -DBColumnNames -ExcludeCustomAnnotations"

Write-Host $Arguments

$Process = Start-Process -FilePath ".\RVTools.exe" -ArgumentList $Arguments -NoNewWindow -Wait -PassThru

if($Process.ExitCode -eq -1)
{
    Write-Host "Error: Export failed! RVTools returned exitcode -1, probably a connection error! Script is stopped" -ForegroundColor Red
    exit 1
}

$OutputFile = $XlsxFileoutput

# =====================================
# RVTools Send Mail
# =====================================

set path=%path%;"C:\Program Files (x86)\Robware\RVTools"

[string] $SMTPserver="relay-mail-server-ip"
[string] $SMTPport="25"
[string] $Mailto="emrahuludag@gmail.com"
[string] $Mailto="group-mail@domain.local"
[string] $Mailfrom="rvtools.report@domain.local"
[string] $Mailsubject="RVTools Inventory Report"
[string] $AttachmentDir="C:\rvtools\"
[string] $XlsxDir = "C:\RVTools\"
[string] $XlsxFile1 = "A-site-vmwarename.xlsx"
[string] $XlsxFile2 = "B-site-vmwarename.xlsx"

# =====================================
# Start RVTools Merge All files
# =====================================

$inputFiles = "$XlsxDir\$XlsxFile1;$XlsxDir\$XlsxFile2"

.\RVToolsMergeExcelFiles.exe -input "$inputFiles" -output "C:\rvtools\Rvtools_merged.xlsx" -overwrite -verbose

# ====================================
# Sending Mail
# ====================================
.\rvtoolssendmail.exe /smtpserver $SMTPserver /smtpport $SMTPport /mailto $Mailto /mailfrom $Mailfrom /mailsubject "RVTools Inventory Report Merged" /attachment C:\rvtools\Rvtools_merged.xlsx
.\rvtoolssendmail.exe /smtpserver $SMTPserver /smtpport $SMTPport /mailto $Mailto /mailfrom $Mailfrom /mailsubject "RVTools Inventory A-site-vmwarename.xlsx Report " /attachment $XlsxDir\$XlsxFile1
.\rvtoolssendmail.exe /smtpserver $SMTPserver /smtpport $SMTPport /mailto $Mailto /mailfrom $Mailfrom /mailsubject "RVTools Inventory B-site-vmwarename.xlsx Report " /attachment $XlsxDir\$XlsxFile2

```

# OUTPUT

```bash
PS C:\Program Files (x86)\Robware\RVTools> C:\Users\euludag\Desktop\RVTools_script\rvtools_export_script.ps1
Start export for vCenter A-site-vmwarename
-u username@vsphere.local -p XXXX -s 10.2.A-site-vmwarename.xlsx.100 -c ExportvInfo2xlsx -d C:\RVTools -f A-site-vmwarename.xlsx -DBColumnNames -ExcludeCustomAnnotations
Start export for vCenter B-site-vmwarename
-u username@vsphere.local -p XXXXX -s B-site-vmwarename.xlsx -c ExportvInfo2xlsx -d C:\RVTools -f B-site-vmwarename.xlsx -DBColumnNames -ExcludeCustomAnnotations

C:\Program Files (x86)\Robware\RVTools
Check input C:\RVTools\\A-site-vmwarename.xlsx
Ok
Check input C:\RVTools\\B-site-vmwarename.xlsx
Ok
05:05:17 Copy C:\RVTools\\kzalavmmgmt.xlsx to C:\rvtools\Rvtools_merged.xlsx
05:05:17 Open XLWorkbook C:\rvtools\Rvtools_merged.xlsx
05:05:18 Open XLWorkbook C:\RVTools\\A-site-vmwarename.xlsx
05:05:18 Merge XLWorkbooks
05:05:18 Processing worksheet vInfo from C:\RVTools\\B-site-vmwarename.xlsx
05:05:18 Processing worksheet vMetaData from C:\RVTools\\B-site-vmwarename.xlsx
05:05:18 Dispose XLWorkbook C:\RVTools\\B-site-vmwarename.xlsx

Mail send to: emrahuludag@gmail.com
RVToolsSendMail: Terminated normally.
Mail send to: emrahuludag@gmail.com
RVToolsSendMail: Terminated normally.

PS C:\Program Files (x86)\Robware\RVTools> 
```


Tüm envanteri topladık ama istediğimiz alanlar olsun sadece isterseniz;
RVTools’tan alınmış birden fazla Excel dosyasını aynı klasöre atıp bu script’i çalıştırıyorsunuz.

Script, sadece belirttiğiniz alanları filtreleyerek tüm verileri birleştiriyor ve size temiz, sade ve odaklanmış bir dosya sunuyor.

Böylece envanterden “işe yarayan” veriye hızlıca ulaşabilirsiniz.

> Gereksinimler : pip install pandas openpyxl

```bash
import pandas as pd
import os

print("Current Working Directory:", os.getcwd())

input_dir = "./exports"

# Create the directory if it doesn't exist
if not os.path.exists(input_dir):
    os.makedirs(input_dir)
    print(f"Created folder: {input_dir}")

# List all .xlsx files in the current directory
files = [f for f in os.listdir('.') if f.endswith('.xlsx')]

all_data = []

# Define the columns we want to extract
wanted_columns = {
    "vInfo": [
        "vInfoVMName",     
        "vInfoPowerstate",           
        "vInfoGuestHostName",            
        "vInfoPrimaryIPAddress",     
        "vInfoNetwork1",   
        "vInfoOS",            
        "vInfoOSTools",      
        "vInfoVISDKServer",        
        "vInfoDataCenter",             
        "vInfoCluster"      
    ]
}

# Process each file
for file in files:
    print(f"Processing {file}")
    xl = pd.ExcelFile(file)
    
    for sheet, columns in wanted_columns.items():
        if sheet in xl.sheet_names:
            df = xl.parse(sheet)
            available_columns = [col for col in columns if col in df.columns]
            
            missing_columns = set(columns) - set(available_columns)
            if missing_columns:
                print(f"Alert: {file} is missing columns: {missing_columns}")
            
            if available_columns:
                df_filtered = df[available_columns].copy()
                df_filtered["source_file"] = file
                all_data.append(df_filtered)
            else:
                print(f"{file} has no matching columns in sheet '{sheet}'.")

# If data is available, create the output files
if all_data:
    final_df = pd.concat(all_data, ignore_index=True)
    
    # Save as Excel
    output_xlsx = os.path.join(input_dir, "rvtools_export.xlsx")
    final_df.to_excel(output_xlsx, index=False)

    # Save as CSV
    output_csv = os.path.join(input_dir, "rvtools_export.csv")
    final_df.to_csv(output_csv, index=False)

    print(f"Exported files:\n- {output_xlsx}\n- {output_csv}")
else:
    print("None of the files contained the desired columns.")

```

# OUTPUT
```bash
PS C:\Users\Desktop\rvtools_export> & C:/Users/Desktop//.venv/Scripts/python.exe "c:/Users/Desktop/rvtools_export/combine_files copy.py"
Current Working Directory: C:\Users\Desktop\rvtools_export
Processing A-site-vmwarename.xlsx
Processing B-site-vmwarename.xlsx
Exported files:
- ./exports\rvtools_export.xlsx
- ./exports\rvtools_export.csv
PS C:\Users\Desktop\rvtools_export
```

```

# OUTPUT
```bash
PS C:\Users\Desktop\rvtools_export> & C:/Users/Desktop//.venv/Scripts/python.exe "c:/Users/Desktop/rvtools_export/combine_files copy.py"
Current Working Directory: C:\Users\Desktop\rvtools_export
Processing A-site-vmwarename.xlsx
Processing B-site-vmwarename.xlsx
Exported files:
- ./exports\rvtools_export.xlsx
- ./exports\rvtools_export.csv
PS C:\Users\Desktop\rvtools_export
```
![](./img/rvtools.png? ':size=80%')