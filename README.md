![MIKES DATA WORK GIT REPO](https://raw.githubusercontent.com/mikesdatawork/images/master/Mikes_Data_Work_Header_890x170.png "Mikes Data Work")        

# SQL-Based-Drive-Space-Alert-With-HTML-Table-Charting
**Post Date: June 4, 2021**


## Contents    
- [About Process](##About-Process)  
- [SQL Logic](#SQL-Logic)  
- [Build Info](#Build-Info)  
- [Author](#Author)  
- [License](#License)      

## About-Process


<p>Here is another example on how to create SQL Based Drive Space Alert With Dynamic HTML Table Charting</p>

![SQL Based Drive Space Alert With HTML Table Charting]( https://mikesdatawork.files.wordpress.com/2021/06/image002.png "SQL Based Drive Space Alert With HTML Table Charting")
 
     
## SQL-Logic
```
use master 
set nocount on
go 
----------------------------------------------------------------------------------------
-- configure database mail and create SQLAlert profile with SMTP server name
 
if (select sum(cast([value_in_use] as int)) from master.sys.configurations where [configuration_id] in ('518', '16386', '16388', '16390'))  <> 4
    begin
        exec master..sp_configure   'show advanced options',		1 reconfigure
        exec master..sp_configure   'Ole Automation Procedures',    1 reconfigure with override
        exec master..sp_configure   'Database Mail XPs',			1 reconfigure with override
		exec master..sp_configure   'xp_cmdshell',					1 reconfigure with override
    end
  
if not exists(select * from msdb.dbo.sysmail_profile where  name = 'SQLAlerts')  
  begin
    execute msdb.dbo.sysmail_add_profile_sp 
      @profile_name = 'SQLAlerts', 
      @description  = 'SQLDatabaseMail'; 
  end
    
  if not exists(select * from msdb.dbo.sysmail_account where  name = 'SQLAlerts@MyCompanyName.com') 
  begin
    execute msdb.dbo.sysmail_add_account_sp 
    @account_name            = 'SQLAlerts@MyCompanyName.com', 
    @email_address           = '', 
    @mailserver_name         = 'mailer.MyCompanyName.com',
    @mailserver_type         = 'SMTP', 
    @port                    = '25', 
    @use_default_credentials =  0 , 
    @enable_ssl              =  0 ; 
  end
    
if not exists(select * 
              from msdb.dbo.sysmail_profileaccount spa
                inner join msdb.dbo.sysmail_profile p on spa.profile_id = p.profile_id 
                inner join msdb.dbo.sysmail_account a on spa.account_id = a.account_id   
              where p.name = 'SQLAlerts' 
                and a.name = 'SQLAlerts@MyCompanyName.com')  
  begin
    execute msdb.dbo.sysmail_add_profileaccount_sp 
      @profile_name = 'SQLAlerts', 
      @account_name = 'SQLAlerts@MyCompanyName.com', 
      @sequence_number = 1; 
  end
 
----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
-- create variables to capture server, instance and port info
 
declare     @instance_name_basic        varchar(255)
declare     @server_name_instance_name  varchar(255)
declare     @server_time_zone			varchar(255)
set			@instance_name_basic        = (select cast(serverproperty('servername') as varchar(255)))
set			@server_name_instance_name  = (select upper(@@servername))
declare     @statements         nvarchar(max)
declare     @original           int
declare     @iid                int
declare     @maxid              int
declare     @domain             nvarchar(255)
declare     @ports      table ([PortType] nvarchar(180), Port int)
exec        master..xp_regread 'HKEY_LOCAL_MACHINE', 'SYSTEM\CurrentControlSet\services\Tcpip\Parameters', N'DomaiN',@Domain output
 
----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
-- create and capture all other instance information found on the server
declare     @instances  table
(
    [InstanceID]    int identity(1, 1) not null primary key
,   [InstanceName]  nvarchar(180)
,   [InstanceFolder]nvarchar(50)
,   [StaticPort]    int
,   [DynamicPort]   int
);
 
declare     @xp_msver_platform  table ([id] int,[name] varchar(180),[internalvalue] varchar(50), [charactervalue] varchar (50))
declare     @platform       varchar(10)
insert into @xp_msver_platform exec master..xp_msver platform
select  @platform = (select 1 from @xp_msver_platform where [charactervalue] like '%86%')
if  @platform is null
begin
    insert into @instances ([InstanceName], [InstanceFolder])
    exec master..xp_regenumvalues N'HKEY_LOCAL_MACHINE', N'SOFTWARE\Microsoft\Microsoft SQL Server\Instance Names\SQL';
end
else
Begin
    insert into @instances ([InstanceName], [InstanceFolder])
    exec master..xp_regenumvalues N'HKEY_LOCAL_MACHINE', N'SOFTWARE\Microsoft\Microsoft SQL Server\Instance Names\SQL';
end
  
declare     @key table ([keyvalue] int)
insert into @key
exec master..xp_regread'hkey_local_machine', N'software\wow6432node\microsoft\microsoft sql server\instance names\sql';
select  @original= [keyvalue] from @key
if      @original=1
        insert into @instances ([instancename], [instancefolder])
        exec master..xp_regenumvalues N'hkey_local_machine', N'software\wow6432node\microsoft\microsoft sql server\instance names\sql';
  
select  @maxid = max([instanceid]), @iid = 1
from    @instances
while   @iid <= @maxid
    begin
        delete from @ports
        select      @statements = 'exec xp_instance_regread N''hkey_local_machine'', N''software\microsoft\\microsoft sql server\' 
        + [instancefolder] + '\mssqlserver\supersocketnetlib\tcp\ipall'', N''tcpdynamicports'''
        from        @instances  where [instanceid] = @iid
        insert into @ports  exec sp_executesql @statements
        select      @statements = 'exec xp_instance_regread N''hkey_local_machine'', N''software\microsoft\\microsoft sql server\' 
        + [instancefolder] + '\mssqlserver\supersocketnetlib\tcp\ipall'', N''tcpport'''
        from        @instances  where [instanceid] = @iid
        insert into @ports  exec sp_executesql @statements
        select      @statements = 'exec xp_instance_regread N''hkey_local_machine'', N''software\wow6432node\microsoft\\microsoft sql server\' 
        + [instancefolder] + '\mssqlserver\supersocketnetlib\tcp\ipall'', N''tcpdynamicports'''
        from        @instances  where [instanceid] = @iid
        insert into @ports  exec sp_executesql @statements
        select      @statements = 'exec xp_instance_regread N''hkey_local_machine'', N''software\wow6432node\microsoft\\microsoft sql server\' 
        + [instancefolder] + '\mssqlserver\supersocketnetlib\tcp\ipall'', N''tcpport'''
        from        @instances  where [instanceid] = @iid
        insert into @ports  exec sp_executesql @statements
        update si   set [staticport] = p.port, [dynamicport] = dp.port from @instances si
        inner join @ports dp on dp.[porttype] = 'tcpdynamicports' join @ports p on p.[porttype] = 'tcpport'
        where [instanceid] = @iid;
        set @iid = @iid + 1
    end

----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
-- create and capture SQL Version info.
declare @version_info   table
(
    [Server_Name]       varchar(255)
,   [SQL_Instance]      varchar(255)
,   [SQL_Version]       varchar(255)
,   [SQL_Build]			varchar(255)
,   [SQL_Edition]       varchar(255)
)
 
insert into @version_info
select
    'ServerName'      = cast(serverproperty('machinename') as varchar)
,   'SQLInstance'     = cast(upper(@@servername) as varchar)
,   'SQLVersioN'      =   
case
when left(cast(serverproperty('productversioN') as varchar), 4) = '14.0' then 'SQL 2017 ' + cast(serverproperty('productlevel') as varchar)
when left(cast(serverproperty('productversioN') as varchar), 4) = '13.0' then 'SQL 2016 ' + cast(serverproperty('productlevel') as varchar)
when left(cast(serverproperty('productversioN') as varchar), 4) = '12.0' then 'SQL 2014 ' + cast(serverproperty('productlevel') as varchar)
when left(cast(serverproperty('productversioN') as varchar), 4) = '11.0' then 'SQL 2012 ' + cast(serverproperty('productlevel') as varchar)
when left(cast(serverproperty('productversioN') as varchar), 4) = '10.5' then 'SQL 2008 R2 ' + cast(serverproperty('productlevel') as varchar)
end
,   'SQLBuild'        = cast(serverproperty('productversioN') as nvarchar(25))
,   'SQLEditioN'      = cast(serverproperty('editioN') as varchar)
 
----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
-- create and capture drive space

declare			
	@drive					varchar(20)
,	@get_fsutil_info		varchar(max)

if object_id('tempdb..#volumes') is not null
	drop table	#volumes
create table	#volumes ([volume] varchar(500)) 
insert into		#volumes 
exec master..xp_cmdshell 'mountvol'
delete #volumes where [volume] is null 
delete #volumes where [volume] like		'%vol%'
delete #volumes where [volume] like		'%cycle%'
delete #volumes where [volume] not like	'%:%'

-- create #drives table from fsutil volume info
if object_id('tempdb..#drives') is not null
	drop table	#drives
create table	#drives ([drive] varchar(500),	[Info] varchar(80)) 
 
declare	get_drive_info cursor for select ltrim(rtrim([volume])) from #volumes 
open	get_drive_info 
fetch next from get_drive_info into @drive 
while @@fetch_status=0  
begin
	set		@get_fsutil_info = 'exec master..xp_cmdshell ''fsutil volume diskfree ' + @drive + '''' 
	insert	#drives ([info]) 
	exec	(@get_fsutil_info) 
	update	#drives set [drive] = @drive where [drive] is null 
	fetch next from get_drive_info into @drive 
end
close	get_drive_info
deallocate get_drive_info

-- get drive space info across drives
if object_id('tempdb..#drive_info') is not null
	drop table	#drive_info

select
	[drive]
,	[totalsize] = sum(case when [info] like 'total # of bytes             : %' then cast(replace(substring([info], 32, 48), char(13), '') as bigint) else cast(0 as bigint) end)
,	[freespace] = sum(case when [info] like 'total # of free bytes        : %' then cast(replace(substring([info], 32, 48), char(13), '') as bigint) else cast(0 as bigint) end) 
into #drive_info from (select [drive], [info] from #drives where [info] like 'total # of %') as D 
group by [drive] 
order by [drive] 


-- get drive space info across drives
declare @drive_space table
(
	[drive]			varchar(3)
,	[total_size]	varchar(20)
,	[used]			varchar(20)
,	[%_full]		varchar(20)
,	[free_space]	varchar(20)
,	[capacity]		varchar(20)
,	[percent]		varchar(20)
,	[percent_diff]	varchar(20)
,	[browse]		varchar(255)
)

declare @alert_metric_capacity	int
set		@alert_metric_capacity	= 95

insert into @drive_space
select 
	'drive' = [drive]
,	'total_size' = 
		case 
			when [totalsize]/1024/1024/1024 = 0 then cast([totalsize]/1024/1024 as varchar) + ' MB'
			when [totalsize]/1024/1024/1024 > 1024 then cast([totalsize]/1024/1024/1024/1024 as varchar) + ' TB'
			else cast([totalsize]/1024/1024/1024 as varchar) + ' GB' 
		end
,	'used' = 
		case 
			when ([totalsize]/1024/1024/1024 - [freespace]/1024/1024/1024) = 0 then cast(([totalsize]/1024/1024 - [freespace]/1024/1024) as varchar) + ' MB'
			when ([totalsize]/1024/1024/1024 - [freespace]/1024/1024/1024) > 1024 then cast(([totalsize]/1024/1024/1024/1024 - [freespace]/1024/1024/1024/1024) as varchar) + ' TB'
			else cast(([totalsize]/1024/1024/1024 - [freespace]/1024/1024/1024) as varchar) + ' GB' 
		end
,	'% full'		= cast(cast(round(100.0 * ([totalsize] - [freespace]) / [totalsize], 0) as decimal(5)) as varchar) + '% Full'
,	'free_space' = 
		case 
			when [freespace]/1024/1024/1024 = 0 then cast([freespace]/1024/1024 as varchar) + ' MB' 
			when [freespace]/1024/1024/1024 > 1024 then cast([freespace]/1024/1024/1024/1024 as varchar) + ' TB' 
			else cast([freespace]/1024/1024/1024 as varchar) + ' GB' 
		end
,	'capacity'		= case when (100.0 * ([totalsize] - [freespace]) / [totalsize]) > @alert_metric_capacity then 'Warning: Low Space' else 'Ok' end
,	'percent'		= cast(cast(round(100.0 * ([totalsize] - [freespace]) / [totalsize], 0) as decimal(5)) as varchar)
,	'percent_diff'	= 100 - (cast(cast(round(100.0 * ([totalsize] - [freespace]) / [totalsize], 0) as decimal(5)) as varchar))
,	'browse'		= '  \\' + cast(serverproperty('MachineName') as nvarchar) + '.' + lower(@domain) + '\' + left([drive], 1) + '$\  '
from 
	#drive_info
order by
	[drive] asc

select * from @drive_space
----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
-- create email warning message

declare @low_drives	varchar(max)
set		@low_drives = ''
select	top 1  @low_drives = @low_drives + 
'(' + left([drive], 1) + ') is ' + [%_Full] + '   ' 
from	@drive_space where [percent] > 89 order by [percent] desc

----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
-- create HTML\xml variables

declare	@message_body			nvarchar(max)
declare @html_block_begin		nvarchar(1000)
declare	@html_block_version		nvarchar(max)
declare @html_block_instance	nvarchar(max)
declare @html_block_drives		nvarchar(max)
declare @html_block_end			nvarchar(1000)

declare @xml_version_info       nvarchar(max)
declare @xml_instance_info      nvarchar(max)
declare @xml_drive_info			nvarchar(max)

----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
-- set XML version_info
 
set @xml_version_info = 
    cast(
        (select
            [Server_Name]       as 'td'
        ,   ''
        ,   [SQL_Instance]      as 'td'
        ,   ''
        ,   [SQL_Version]       as 'td'
        ,   ''
        ,   [SQL_Build]			as 'td'
        ,   ''
        ,   [SQL_Edition]       as 'td'
 
        from @version_info 
        order by [SQL_Instance] asc 
        for xml raw('tr')
    ,   elements, type)
    as nvarchar(1000)
        )
 
----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
-- set XML for instance_info
set @xml_instance_info = 
    cast(
        (select
            cast(serverproperty('MachineName') as nvarchar) + '.' + lower(@domain) +  + case when [InstanceName] = 'MSSQLServer' then '' else '\' + [InstanceName] end
        +   case when [StaticPort] is null then ',' + cast([DynamicPort] as varchar) else ',' + cast([StaticPort] as varchar) end as 'td'
        ,   ''
        ,    cast(serverproperty('ComputerNamePhysicalNetBIOS') as varchar)     as 'td'
        ,   ''
        ,    [InstanceName]														as 'td'
        ,   ''
        ,    isnull(cast([StaticPort] as varchar), '')							as 'td'
        ,   ''
        ,    isnull(cast([DynamicPort] as varchar), '')							as 'td'
 
        from @instances 
        order by [InstanceName] asc 
        for xml raw('tr')
    ,   elements, type)
    as nvarchar(2000)
        )

----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
-- set xml for drive info
set @xml_drive_info = 
    cast(
        (select 
            [drive]				as 'td'
        ,   ''
        ,   [total_size]		as 'td'
        ,   ''
        ,   [used]				as 'td'
        ,   ''
		,	[%_full]			as 'td'
		,	''
		,	case 
				when [percent] > 89 then cast('<table style="border-collapse: collapse; border: none;" cellpadding="0" cellspacing="0" width="250"><td bgcolor="#C70039" style="width:' + cast([percent] as varchar) + '%; border: none; background-color:#C70039; float:left; height:12px"></td><td bgcolor="#1D1D1D" style="width:' + cast([percent_diff] as varchar) + '%; border: none; background-color:#1D1D1D; float:left; height:12px;"></td></table>' as xml)
				else cast('<table style="border-collapse: collapse; border: none;" cellpadding="0" cellspacing="0" width="250"><td bgcolor="#CA9321" style="width:' + cast([percent] as varchar) + '%; border: none; background-color:#CA9321; float:left; height:12px"></td><td bgcolor="#1D1D1D" style="width:' + cast([percent_diff] as varchar) + '%; border: none; background-color:#1D1D1D; float:left; height:12px;"></td></table>' as xml) 
			end as 'td'
		,	''
		,	[browse]			as 'td'
		,	''
		from @drive_space
		order by [drive] asc
        for xml raw('tr')
    ,   elements, type)
    as nvarchar(max)
        )


----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
-- create html message blocks

set @html_block_begin =
'<html><head><style>
BODY {background-color:#141414; line-height:1px; -webkit-text-size-adjust:none; color: #ccc; font-family: sans-serif;}
        H1 {font-size: 90%; color: #bbb;}
        H2 {font-size: 90%; color: #bbb;}
        a {color: #19A8F6; text-decoration: none;}
        TABLE, TD, TH {
            font-size: 87%;
            border: 1px solid #bbb;
            border-collapse: collapse;
        }                         
        TH {
            font-size: 87%;
            text-align: left;
            background-color: #1A1B20;
            color: #f8ab0c;
            padding: 4px;
            padding-left: 7px;
            padding-right: 7px;
        }
		TD {
            font-size: 87%;
            padding: 4px;
            padding-left: 7px;
            padding-right: 7px;
            max-width: 100px;
            overflow: hidden;
            text-overflow: ellipsis;
            white-space: nowrap;
        }

</style></head><body>'

set @html_block_version = 
'
<p style="font-family:sans-serif; font-size:20px; color:#f8ab0c;">SQL Alert - Drive Space</p>
<p style="color:#f8ab0c;">Database Server: ' + @server_name_instance_name + '    ' + @low_drives + '</p>
<hr style="border: none; height: 2px; color: #ca9321; background: #ca9321;">
<br>
<p>SQL Version</p>
<table border="1">
        <tr>
            <th> SERVER NAME  </th>
            <th> SQL INSTANCE </th>
            <th> SQL VERSION  </th>
            <th> SQL BUILD </th>
            <th> SQL EDITION  </th>
        </tr>'        
+ @xml_version_info + '</table>
<br>
'

set @html_block_instance = 
'
<p>SQL Instances On This Server</p>
<table border="1">
        <tr>
            <th> INSTANCE CONNECTION STRING </th>
            <th> NET BIOS NAME </th>
            <th> INSTANCE NAME </th>
            <th> STATIC PORT </th>
            <th> DYNAMIC PORT </th>
        </tr>'        
+ @xml_instance_info + '</table>
<br>
'

set @html_block_drives = 
'
<p>Drives</p>
<table border="1">
        <tr>
            <th> DRIVE </th>
            <th> TOTAL SIZE </th>
            <th> USED </th>
			<th> % FULL </th>
			<th> GRAPH </th>
			<th> BROWSE </th>
        </tr>'
  
+ @xml_drive_info + '</table>
<br>

<p>Get a list of SQL Versions at the<a href="https://sqlserverbuilds.blogspot.com/">Server Builds Blogspot</a>.</p>
'

set @html_block_end =
'<br></body></html>'

----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
-- combine HTML formatted blocks

set @message_body =
	@html_block_begin
+	@html_block_version
+	@html_block_instance
+	@html_block_drives
+	@html_block_end

----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
-- set message subject.

declare @message_subject    varchar(1000)
set		@message_subject = 'SQL Drive Alert on [' + @instance_name_basic + ']... ' + @low_drives
 
----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
-- send email.

if exists(select top 1 [percent] from @drive_space where [percent] > 89)
    begin
        exec    msdb.dbo.sp_send_dbmail
                @PROFILE_NAME   = 'SQLAlerts'
            ,   @RECIPIENTS		= 'MyUserName@MyCompanyName.com'
            ,   @BODY			= @message_body 
            ,   @BODY_FORMAT    = 'HTML';
    end


```

[![WorksEveryTime](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://shitday.de/)

## Build-Info

| Build Quality | Build History |
|--|--|
|<table><tr><td>[![Build-Status](https://ci.appveyor.com/api/projects/status/pjxh5g91jpbh7t84?svg?style=flat-square)](#)</td></tr><tr><td>[![Coverage](https://coveralls.io/repos/github/tygerbytes/ResourceFitness/badge.svg?style=flat-square)](#)</td></tr><tr><td>[![Nuget](https://img.shields.io/nuget/v/TW.Resfit.Core.svg?style=flat-square)](#)</td></tr></table>|<table><tr><td>[![Build history](https://buildstats.info/appveyor/chart/tygerbytes/resourcefitness)](#)</td></tr></table>|

## Author

[![Gist](https://img.shields.io/badge/Gist-MikesDataWork-<COLOR>.svg)](https://gist.github.com/mikesdatawork)
[![Twitter](https://img.shields.io/badge/Twitter-MikesDataWork-<COLOR>.svg)](https://twitter.com/mikesdatawork)
[![Wordpress](https://img.shields.io/badge/Wordpress-MikesDataWork-<COLOR>.svg)](https://mikesdatawork.wordpress.com/)

     
## License
[![LicenseCCSA](https://img.shields.io/badge/License-CreativeCommonsSA-<COLOR>.svg)](https://creativecommons.org/share-your-work/licensing-types-examples/)

![Mikes Data Work](https://raw.githubusercontent.com/mikesdatawork/images/master/Mikes_Data_Work_Social.png "Mikes Data Work")
