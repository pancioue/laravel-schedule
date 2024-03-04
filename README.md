# laravel 排程schedule

有關withoutOverlapping
實測如下(測試機docker)
測試一.  
a file:  
  Log::info('aaa');  
  sleep(10);  
b file:  
  Log::info('bbb');  
  sleep(1000);  
c file  
  Log::info('ccc');  
  sleep(100);  

kernel:
```
$schedule->command('suppliers:test-a')->everyMinute();
$schedule->command('suppliers:test-b')->everyMinute()->withoutOverlapping();
$schedule->command('suppliers:test-c')->everyMinute()->withoutOverlapping();
```

結果如下
> [2024-01-12 17:37:01] local.INFO: aaa第一次a  
[2024-01-12 17:37:12] local.INFO: bbb第一次b  
[2024-01-12 17:38:02] local.INFO: aaa第二次a  
[2024-01-12 17:38:12] local.INFO: ccc第二次c  
[2024-01-12 17:39:01] local.INFO: aaa第三次a，第一輪b還在執行跳過，第二輪c還在執行跳過，第三輪結束  
在這之間第二次c sleep100秒結束，第二輪就此結束  
[2024-01-12 17:40:01] local.INFO: aaa第四次a  
[2024-01-12 17:40:12] local.INFO: ccc第四次c  

=> 小結: 第一次執行因為要等1000秒，卡在b就停了，但C不會因此卡住，下一分鐘依然會直接執行C  


測試二.  
d file:  
  Log::info('ddd');  
  sleep(10);  
e file:  
  Log::info('eee');  
  sleep(55);  
f file  
  Log::info('fff');  
  sleep(100);  

kernel:
```
$schedule->command('suppliers:test-d')->everyMinute();
$schedule->command('suppliers:test-e')->everyMinute()->withoutOverlapping();
$schedule->command('suppliers:test-f')->everyMinute()->withoutOverlapping();
```

結果如下  
> [2024-01-12 18:02:01] local.INFO: ddd第一次的d  
[2024-01-12 18:02:11] local.INFO: eee第一次的e  
[2024-01-12 18:03:02] local.INFO: ddd第二次的d  
[2024-01-12 18:03:07] local.INFO: fff第一次的f  
[2024-01-12 18:03:12] local.INFO: eee第二次的e  
第二次的e加55秒會超過到下分鐘，但下一次的e還是有印出來，推斷應該是執行階段才會判斷使否正在執行  
[2024-01-12 18:04:02] local.INFO: ddd第三次的d  
[2024-01-12 18:04:12] local.INFO: eee第三次的e  
第一次的f經過100秒，第一輪結束  
[2024-01-12 18:05:01] local.INFO: ddd第四次的d  
[2024-01-12 18:05:07] local.INFO: fff第三次的f  
[2024-01-12 18:05:11] local.INFO: eee第四次的e  


### runInBackground() 可以併發執行  
ex.
```
for ($i = 0; $i < 2; $i += 1) {
    $schedule->command('suppliers:renew-amazon-products')->everyMinute()->runInBackground(); // 兩次一起跑

    $schedule->command('suppliers:renew-amazon-products' . $i)->everyMinute()->withoutOverlapping(); // 一個一個跑但不會重複跑

    //可以阻止同程序名稱重複跑，並同時發送，可以將每分鐘控制在想要的線程數
    $schedule->command('suppliers:renew-amazon-products' . $i)->everyMinute()->withoutOverlapping()->runInBackground();
}
```

### 結論
想要最大化cpu使用率，又不想因為一支跑太久塞車跑導致cpu爆掉，可以使用  
`->withoutOverlapping()->runInBackground();` 控制在固定的線程數
