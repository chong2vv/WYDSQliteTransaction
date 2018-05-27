# WYDSQliteTransaction
在IM中大多数消息都是存储于移动端，如果单条插入的话，频繁的开锁库效率极低，所以往往会用到批插。但是对于移动端来说，批插是很少用到的，所以在这里贴一下之前做IM时候批量拉取消息插入时的写法（事物）。

```
	//存储每条插入语法的数组
	NSMutableArray *transactionSql = [NSMutableArray array];

	//此处数组内的元素就是没调要插入数据的插入语法
    for (ConvModule * convModule in array) {
    
     NSString *insertSQL = [NSString stringWithFormat:@"INSERT INTO %@ (convid,position,convname,latestupdateTime,latestupdateusername,latestupdatemessage) VALUES(\"%@\",%d,\"%@\",%lli,\"%@\",\"%@\")",TB_CONV_LIST, convModule.convId, 0 , convModule.convName, convModule.latestUpdateTime, convModule.latestUpdateUserName, convModule.latestUpdateMessage];
        
        [transactionSql addObject:insertSQL];
    }
    
    sqlite3 *sqlitedb = [self openSqlite];
    if (!sqlitedb) return;
	   
    @try{
        char *errorMsg;
        //启动事物
        if (sqlite3_exec(sqlitedb, "BEGIN TRANSACTION", NULL, NULL, &errorMsg)==SQLITE_OK)
        {
            NSLog(@"启动事务成功");
            sqlite3_free(errorMsg);
            sqlite3_stmt *statement;
            for (int i = 0; i<transactionSql.count; i++)
            {
                const char *insert_stmt = [[transactionSql objectAtIndex:i] UTF8String];
                if (sqlite3_prepare_v2(sqlitedb,insert_stmt, -1, &statement,NULL)==SQLITE_OK)
                {
                    if (sqlite3_step(statement)!=SQLITE_DONE) sqlite3_finalize(statement);
                }
            }
            
            int nRet=0;
            
            int breakCount = 0;
            do
            {
                nRet = sqlite3_exec( sqlitedb ,"COMMIT" , NULL , NULL , &errorMsg );
                
                //这里其实比较玄学，主要是为了在commit之后查一下库有没有死锁。如果第一次查询被锁死了，那么会延时一秒后再继续查询，直到最多四次（因为目前来看，如果四次还没有解锁的话，基本上就可以确定已经锁死，那么这个时候只能抛出异常，回滚。）
                if (nRet == SQLITE_BUSY)
                {
                    sleep(1);
                    
                    breakCount ++;
                    if (breakCount > 4) {
                        break;
                    }
                    continue;
                }
                break;
            } while (1);

            sqlite3_free(errorMsg);
        }
        else sqlite3_free(errorMsg);
    }
    
    @catch(NSException *e)
    {
    
        char *errorMsg;
        if (sqlite3_exec(sqlitedb, "ROLLBACK", NULL, NULL, &errorMsg)==SQLITE_OK)  NSLog(@"回滚事务成功");
        sqlite3_free(errorMsg);
    }
    
    @finally{}
    sqlite3_close(sqlitedb);

```

