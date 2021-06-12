# Bukkit编程杂谈：雷点|模板|思路|设计

by 南外丶仓鼠

[TOC] 

## 一、雷点|常见问题处理方案|必须写的代码

------

### ○ InventoryClickEvent的注意事项

在使用InventoryClickEvent时，不免会用到getClickedInventory()和getInventory()函数

这两个函数有很大差别，无论何时，**使用getClickedInventory()总比getInventory()保险**：

> getInventory()表示玩家**额外打开的Inventory**，即位于玩家**背包上方的Inventory**
>
> getClickedInventory()表示玩家**当前点击的格子所在的Inventory**

同时地，玩家不一定点击的是格子，有可能点击GUI之外的区域，导致getInventory()有可能为null。

此时需要先**判定getInventory()是否为null**，以免后续调用的时候出现NPE错误

不过有一些注意点，点击格子之外时：

> 若点击的是Gui外的灰色区域：
>
> getInventory()不为null，这是显而易见的，它会返回玩家打开的Inventory，**点击的slot为999**
>
> getClickedInventory()为null，毕竟玩家点击了**本没有Inventory的位置**
>
> 若点击的是Gui内非格子区域，即边框区域：
>
> **不会触发InventoryClickEvent**

```java
@EventHandler
    public void onClick(InventoryClickEvent event) {
        System.out.println("触发了InventoryClickEvent事件！");
        if (event.getInventory() == null) {
            System.out.println("getInventory()为null");
        } else {
            System.out.println("getInventory()的容量" + event.getInventory().getSize());
        }
        if (event.getClickedInventory() == null) {
            System.out.println("getClickedInventory()为null");
        } else {
            System.out.println("getClickedInventory()的容量" + event.getClickedInventory().getSize());
        }
        System.out.println("点击的格子ID为" + event.getSlot());
    }
```



### ○ 监听事件与优先度的注意事项

假设某个监听器监听了BlockBreakEvent，意在在玩家放置泥土方块时把这个方块变成钻石方块。但是如果这个玩家在别的玩家的领地里放置泥土方块，被Residence插件阻止了，上述的监听器仍会继续工作。

此时需要**先判定BlockBreakEvent#isCancelled()**，如果已经被取消了，则不执行代码

Bukkit提供了优先度系统以让开发者们更好地实现上面的功能，**在@EventHandler注解处增加(priority=EventPriority.优先度等级)可为监听级设置优先度**，等级**越低越先执行**，即执行顺序为：

> LOWEST>LOW>NORMAL>HIGH>HIGHEST>MONITOR

因此，别的插件和你的插件可能会监听**相同的事件**，如果你的监听器需要用到**setCancelled(true)**，请**降低**你的监听器的优先度，让事件触发时被**阻止得更快**，以免其他插件出现错误。

> 简单来说，我们一般认为在优先度为EventPriority.HIGH以上的监听器中setCancelled(true)的行为是**自取灭亡**，这会造成别的插件和玩家的困扰！建议设置这种监听器的优先度为**EventPriority.LOW及以下**。因为有很多开发者懒得标注priority，导致他们的监听器优先度为**EventPriority.NORMAL**，此时可能会发生已经取消的事件触发了**本不应该触发的代码**！
>
> 同样地，如果你的监听器中包含**提供后续效果、增添机制的代码**，请一定要**判定isCancelled()**，并把优先度设置到**EventPriority.HIGH以上**，毕竟有很多开发者懒得标注priority，他们有可能在优先度为EventPriority.NORMAL的监听器中setCancelled(true)，而你是无法改变这点的。

至于同优先度时，事件是以什么顺序触发的呢：

**同优先度的事件触发顺序为它们的注册顺序（from 果粒橙姐姐）**



### ○ Bukkit.getPlayer(UUID uuid)优于Bukkit.getPlayer(String name)

**尽量通过玩家的UUID而不是名字**获取服务器玩家，原因有二：

> 1.getPlayer(String name)会造成**服务器负担**（虽然不大，但是仍存在）
>
> 2.正版玩家可以修改名字

不过有时候的确需要Bukkit.getPlayer(String name)，例如**指令参数为玩家名**时

另外如果获取不到玩家，**返回null**，注意后续的特判防止NPE

**至于Bukkit.getOfflinePlayer(String name)则不是很被忌讳，毕竟盗版uuid可以直接算出来，在线uuid从Mojang API/Yggdrasill API/特殊插件设置获取（from 星空姐姐、MiaoLio）**

**对于多有Yggdrasill的服务器则需要另作处理（from 果粒橙姐姐）**

Yggdrasill API相关资料https://printempw.github.io/minecraft-yggdrasil-api-third-party-implementation/



### ○ 载入时或重载时需要注意的事情

很多时候，服主们都是把插件扔进plugins，在服务器开启的情况下**使用PlugMan或Yum这样的管理型插件**来启用你的插件，如果你的插件没有自带reload，他们一般**也会使用管理插件来重载你的插件**，而不是/reload或重启。

这个过程跳过了**/reload和/restart正常该走的流程**，以至于你的插件载入或重载时**面临了一些问题**。

例如已在服务器内的玩家**信息未载入**，解决方案是在每次启用插件时载入目前在线玩家的数据：

```java
for(Player player:Bukkit.getOnlinePlayers()){
	//载入玩家信息的代码
}
```

同时地，如果你要自己写reload指令，不要**无脑来一遍onDisable()、onLoad()和onEnable()**，注意**线程、监听器注册等的问题**

> 另一个比较**不受欢迎**的处理方案是**拒绝reload**，例如MCMMO，非重启服务器不能加载：这很安全，不过会导致使用的不便，以至于MCMMO不能重载的尿性成为服主们的饭后谈资。



### ○ 异步不能用Bukkit API

很多开发者在追求线程安全的情况下，会**选择异步处理**。

但是一般情况下异步处理时**不能调用BukkitAPI**，**不过有时候可以（from 星空姐姐）**

此时的解决方案是在你的异步中**再套一个非异步的BukkitRunnable**（虽然我经常对此感觉不安）

> 如果你的控制台出现以下报错，则大概率是出现这个问题了
>
> java.lang.IllegalStateException: Asynchronous xxx update!



### ○ 不可监听AbstractEvent

> 曾今有人提出了这个问题：禁止一个玩家在服务器的所有举动（例如未登录时），直接监听关于该玩家的PlayerEvent，如果是Cancellable的，直接setCancelled(true)不就行了？

这样的想法是想peach，不仅PlayerEvent，BlockEvent、EntityEvent都不可以这样监听。

简而言之，**有abstract关键字修饰**的事件都不能监听！Bukkit API只允许我们监听**详细的、实在的事件**！

同样的，你在为自己和别的开发者**创建自定义事件**时，**注意abstract的使用**，它直接影响的自定义事件的可监听性。



### ○ Location、ItemStack操作时记得clone，而不是直接修改

这个是Java常识！

例如Location#add(x,y,z)返回的是**修改过的自己**

同样的，你先给予玩家一个ItemStack，事后再对其进行修改，**玩家背包的ItemStack会跟着改变**！

因此**尽量先clone在修改**。

> 毕竟万一你事后把玩家背包的物品改成了他本不该有的好东西就麻烦了嘛。



### ○ 请勿将玩家和NPC混为一谈

在NPC在正常使用时和玩家**没有明显的区别**，即Citizens、FakePlayers等插件创建出来的假玩家会被代码检测到，且会被**当做正常玩家处理**。

有时候**对NPC的“数据”修改时会出现报错**，因此每次加载、保存、遍历时**特判NPC是必要的**，不仅为了**防止报错**，还可以**减少能耗**嘛。

以下是特判的代码：

```java
//这里的NMSHelper不放了，比较简单
import ute.nms.NMSHelper;
import net.citizensnpcs.api.npc.NPC;
import net.citizensnpcs.npc.ai.NPCHolder;
import org.bukkit.entity.Entity;

import com.infumia.fakeplayer.api.INPC;

import java.util.LinkedList;
import java.util.function.Predicate;

@SuppressWarnings("Convert2Lambda")
public class NPCChecker {
    private static final LinkedList<Predicate<Entity>> checkers = new LinkedList<>();

    public static void clearAll() {
        checkers.clear();
    }

    public static boolean isNPC(Entity entity) {
        for (Predicate<Entity> checker : checkers) {
            if (checker.test(entity)) return true;
        }
        return false;
    }

    public static void register(Predicate<Entity> checker) {
        checkers.add(checker);
    }

    static {
        try {
            // For Citizens
            Class.forName("net.citizensnpcs.npc.ai.NPCHolder");
            register(new Predicate<Entity>() {
                @Override
                public boolean test(Entity entity) {
                    return entity instanceof NPC || entity instanceof NPCHolder;
                }
            });
        } catch (Throwable ignore) {
        }
        try {
            // For FakePlayer (73139)
            Class.forName("com.infumia.fakeplayer.api.INPC");
            register(new Predicate<Entity>() {
                @Override
                public boolean test(Entity entity) {
                    return NMSHelper.getHandle(entity) instanceof INPC;
                }
            });
        } catch (Throwable ignore) {
        }
    }
}
```



### ○ Map遍历时不要修改

又是一个Java常识，修改正在遍历Map会**报出ConcurrentModificationException**。

实在不行就遍历**clone的Map**，中途有需要再修改原Map。

> 永远记住你可能会犯任何错误，当你发现你的Map在遍历时，另一个远在天边的类中的操作让你的代码报废就已经为时已晚了！



### ○ 避免药水效果赋予失败

赋予药水效果时检测原来**有没有同类型的效果**，如果**有先去除原效果再赋予**

```java
public static void addPotionEffect(Player player,PotionEffect efect){
	if(player.hasPotionEffect(effect.getType())){
		player.removePotionEffect(effect.getType());
	}
	player.addPotionEffect(effect);
}
```

> 不过有趣的是，有一个看起来比较棒的函数：
>
> LivingEntity#addPotionEffect([@NotNull](https://javadoc.io/doc/org.jetbrains/annotations-java5/20.1.0/org/jetbrains/annotations/NotNull.html) [PotionEffect](https://hub.spigotmc.org/javadocs/spigot/org/bukkit/potion/PotionEffect.html) effect, boolean force)
>
> 可以强制覆盖原有药水效果，但不知道为什么被**标记为过时**了。



### ○ 玩家加入时的处理

玩家进入服务器时对其操作要谨慎，如果是插件自身数据的加载就罢了，若**对玩家的坐标、背包等数据通过Bukkit API进行修改，则需要等一会**。

为什么呢？因为PlayerJoinEvent在触发时**玩家还没有彻底载入完毕**。

我们一般使用**BukkitRunnable延迟几刻**再进行处理：

```java
@EventHandler 
public void onJoin(PlayerJoinEvent event){
	Player player=event.getPlayer();
	Location loc=new Location(player.getWorld(),0,72,0);
	new BukkitRunnable(){
		@Override
		public void run(){
			event.getPlayer().teleport(loc);
		}
	}.runTaskLater(this,3L); //延迟
}
```



### ○ 空气物品没有ItemMeta

空气物品**始终没有ItemMeta**，对其getItemMeta()**始终返回null**。

即使把由内容的ItemMeta赋予空气物品，也是**无济于事**。

而有些物品则具有**更详细的ItemMeta**，后面会详细介绍。

例如SkullItem，此时将其ItemMeta**向下转型**可以得到其SkullMeta。



### ○ 差之毫厘，谬以千里

Bukkit.getPluginManager().registerEvents([@NotNull](https://javadoc.io/doc/org.jetbrains/annotations-java5/20.1.0/org/jetbrains/annotations/NotNull.html) [Listener](https://hub.spigotmc.org/javadocs/spigot/org/bukkit/event/Listener.html) listener, [@NotNull](https://javadoc.io/doc/org.jetbrains/annotations-java5/20.1.0/org/jetbrains/annotations/NotNull.html) [Plugin](https://hub.spigotmc.org/javadocs/spigot/org/bukkit/plugin/Plugin.html) plugin)

Bukkit.getPluginManager().registerEvent([@NotNull](https://javadoc.io/doc/org.jetbrains/annotations-java5/20.1.0/org/jetbrains/annotations/NotNull.html) [Class](https://docs.oracle.com/en/java/javase/16/docs/api/java.base/java/lang/Class.html)<? extends [Event](https://hub.spigotmc.org/javadocs/spigot/org/bukkit/event/Event.html)> event, [@NotNull](https://javadoc.io/doc/org.jetbrains/annotations-java5/20.1.0/org/jetbrains/annotations/NotNull.html) [Listener](https://hub.spigotmc.org/javadocs/spigot/org/bukkit/event/Listener.html) listener, [@NotNull](https://javadoc.io/doc/org.jetbrains/annotations-java5/20.1.0/org/jetbrains/annotations/NotNull.html) [EventPriority](https://hub.spigotmc.org/javadocs/spigot/org/bukkit/event/EventPriority.html) priority, [@NotNull](https://javadoc.io/doc/org.jetbrains/annotations-java5/20.1.0/org/jetbrains/annotations/NotNull.html) [EventExecutor](https://hub.spigotmc.org/javadocs/spigot/org/bukkit/plugin/EventExecutor.html) executor, [@NotNull](https://javadoc.io/doc/org.jetbrains/annotations-java5/20.1.0/org/jetbrains/annotations/NotNull.html) [Plugin](https://hub.spigotmc.org/javadocs/spigot/org/bukkit/plugin/Plugin.html) plugin)

> 源于某人的提问，当时我百思不得其解。



### ○ 传送时会打断骑乘效果

在玩家骑乘时，无论是**传送玩家**，还是**传送坐骑或载具**，均会**打断骑乘状态**。

因此你想移动骑乘中的玩家时，不如试试看对**载具setVelocity(Vector vector)**，实在不行传送后重新设置**setPassenger(Entity entity)**（造成游戏体验不良好）



### ○ setTarget()只适用于攻击性生物

> 不要被Bukkit的JavaDoc骗了！

```java
Mob#setTarget(Entity entity)

//Instructs this Mob to set the specified LivingEntity as its target.
//Hostile creatures may attack their target, and friendly creatures may follow their target.

//Parameters:
//target - New LivingEntity to target, or null to clear the target
```

说的好听“friendly creatures may follow their target”，但是使用时**非攻击性生物根本无动于衷**！

这个方法只适用于**攻击性生物**，如果你要让友好的动物朋友们自己寻路，使用**Nevigation**：

```java
//这里我懒得放反射代码了
import org.bukkit.craftbukkit.v1_16_R3.entity.CraftCreature;

public static void walkToLocation(Location location, float speed, Entity entity) {
        try {
            ((CraftCreature) entity)
                    .getHandle()
                    .getNavigation()
                    .a(location.getX(), location.getY(), location.getZ(), speed);
        } catch (Exception exception){
            return;
        }
    }
```

> 我自己在测试时，发现上面这个speed很魔性，它以指数级上升，类似于某个高中函数
>
> 测试得**1.75f约为小僵尸移速**，**1.25f约为正常走路移速**

------

## 二、常用模板|建议|设计规则

### ○ 方块数据的操作与存储：BlockState与BlockData

机制类插件一般都会涉及方块数据的操作。

该如何处理呢？

如果是**原版就有的数据**，例如箱子内的物品，只需要对这个箱子的**BlockState**转成Chest，再对其操作：

```java
Block block; //假设这个block是个箱子 
Chest chest= (Chest) block.getState(); //把BlockState转换成Chest进行操作
Inventory inv=chest.getBlockInventory(); //箱子的Inventory
inv.setItem(0,new ItemStack(Material.TORCH)); //在第一格处放置一个火把
chest.setLock("钥匙"); //为箱子上锁，需要名为“钥匙”的物品才能打开
```

这里可以发现：原版数据标签系统的数据标签与BlockState有联系（当然啊喂！）

> 至于**保存**我们的方块，此时使用**BlockData#getAsString()**就可以了，读取时使用**Bukkit.createBlockData(String dataString)**，再使用**Block#setBlockData(BlockData blockData)**即可。**（1.13+）**
>
> 低版本不存在BlockData这个美妙的轮子，此时需要**存TileEntity**，转成NBT保存（from 果粒姐姐）
>
> ```java
> NBTTagCompound nbt=((CraftChunk) block.getChunk())
> 		.getHandle()
>     	.tileEntities
>     	.get(new BlockPosition(block.getX(),block.getY(),block.getZ()))
>     	.save(new NBTagCompound());
> try{
>     NBTCompressedStreamTools.a(nbt,out);
> }catch(IOException e){
>     e.printStackTrace();
> }
> ```

> 如果要高效的存BlockState得用**注册表序列化成int**（from 海螺螺）

如果是插件自己创造、操控的数据，建议使用Map把序列化的Location和数据对应起来，存进文件中。



### ○ InventoryHolder优于标题

> 在很久很久之前InventoryHolder就已经存在了，但是大家都喜欢用标题去判定一个Inventory。
>
> 直到1.13之后完全去除了直接从Inventory得到title的可能性，InventoryHolder广而应用。

如何使用InventoryHolder？

1️⃣ 自己写一个**实现InventoryHolder**的类，例如：

```java
//类名随意
public static class CustomHolder implements InventoryHolder {
    @Override
    public Inventory getInventory() {
        return null;
    }
}
```

2️⃣ 在创建你自己的Inventory时，owner**填写你的Holder**

```java
Bukkit.createInventory(new CustomHolder(),54,title);
```

3️⃣ **判定**时，只需要这样写就行了：

```java
Inventory inv=event.getInventory();
if(inv.getHolder() instanceof CustomHolder){
	//判定成功后执行的代码
}
```

> 你可以创建很多个Holder，对应**不同需求**的Inventory。



### ○ 组装事件

Bukkit并没有给我们游戏内所有可能的事件，所以我们需要学会"组装"监听器。思路是通过多个监听器**接连触发**进行判定，使用Map存储**需要中转的值**。

例如玩家吃东西的动作可以分为两步，**从前往后**发生：

> 消耗食品*1（PlayerItemConsumeEvent事件）
>
> 饱食度+N（FoodLevelChangeEvent事件）

因此我们分别监听以上两个事件，其中做一些**交接工作**即可。这里在玩家吃东西时输出了**食品名称**：

```java
public static HashMap<UUID, Material> foods = new HashMap<>();
    
@EventHandler
public void onConsume(PlayerItemConsumeEvent event) {
    Player player = event.getPlayer();
    ItemStack item = event.getItem();
    //可食用的物品，而不是其他的stuff
    if (item.getType().isEdible()) {
        foods.put(player.getUniqueId(), item.getType());
    }
}

@EventHandler
public void onEat(FoodLevelChangeEvent event) {
    Player player = (Player) event.getEntity();
    //如果就在刚刚消耗了食物，则百分之一万是吃的东西
    if (foods.containsKey(player.getUniqueId())) {
        player.sendMessage(foods.get(player.getUniqueId()).name());
        foods.remove(player.getUniqueId());
    }
}
```



### ○ PlayerInteractEvent的注意点

如果你经常使用PlayerInteractEvent，会发现玩家点击一次，实际上**触发了两次事件**。

这是为什么呢？因为Bukkit分别触发了**左右手点击**的事件，例如玩家左手持火把，右手持剑，右击泥土方块，Bukkit会分别触发“火把右击泥土“和“剑右击泥土”两个事件。

值得一提的是，即使玩家由一只手是空着的，或者两只手是空着的，**也会触发两次**。

这的确让开发者们困扰且恼火。我们一般使用**PlayerInteractEvent#getHand()**进一步判定左右手：

```java
@EventHandler
public void onClick(PlayerInteractEvent event){
	if(event.getHand()==EquipmentSlot.HAND){
		//右手
	}
	if(event.getHand()==EquipmentSlot.OFFHAND){
		//左手
	}
}
```

与其相似的**PlayerInteractEntityEvent**等也是如此。



### ○ 时间的计算

1️⃣ 冷却时间的处理

一般使用Map存储玩家**上一次操作的时间戳**，下一次使用时进行比对，检测是否**超过冷却时间**：

```java
public static HashMap<UUID,Long> lastUseStamps=new HashMap<>();

//使用时
if(lastUseStamps.containsKey(player.getUniqueId())){
    if(System.currentTimeMillis()-lastUseStamps.get(player.getUniqueId())<cooldown*1000){
        int lasted=cooldown-System.currentTimeMillis()-lastUseStamps.get(player.getUniqueId())/1000;
        player.sendMessage("冷却时间未到，还剩下"+lasted+"秒！");
        return;
    }
}
//使用的代码
lastUseStamps.put(player.getUniqueId(),System.currentTimeMillis());
```

> 开一个Runnable甚至Thread去每秒计算秒数是**绝对浪费能耗**
>
> 不过如果需要可以实时显示、刷新的时间，一般采用后者。

另外，在比较高的版本，你可以使用**HumanEntity#setCooldown(Material type,int ticks)**，这样可以让玩家在一定的时间内不能使用**某个type的所有物品**。（参照末影珍珠冷却的效果）

2️⃣ 世界时间的处理

MC的一天为23000刻，粗略来说：

> **傍晚**开始的时候为14000刻
>
> **深夜**开始的时候为16000刻
>
> **清晨**开始的时候为23000刻

这里提供一个美妙的函数，将世界时间**可理解化**：

```java
//把世界时间刻变为刻度的hh:mm
public static String getFormatTime(long newTime) {
    long hour = newTime / 1000;
    long minute = (long) (newTime % 1000 * 0.06);
    String tmp = "";
    if (minute < 10) {
        tmp += "0" + minute;
    } else {
        tmp += minute;
    }
    if (hour + 6 >= 24) {
        return "0" + (hour + 6 - 24) + ":" + tmp;
    } else {
        return (hour + 6) + ":" + tmp;
    }
}
```



### ○ @EventHandler 的 ignoreCancelled = true

这个巧妙的变量——ignoreCancelled，可以让你的监听器在已经被优先度更低的监听器取消的事件触发时正常执行！

如果ignoreCancelled为true，则监听器**会无视取消**，一丝不苟地触发。

反之，监听器在事件已经被取消时，**保持沉默**。



### ○ 机制类插件的人性化设置

这是设计上的小tricks：

1️⃣ 在配置中允许用户调试**启用的世界和禁用的世界**。

> 毕竟有些服主不太会开群组服，他们的所有“服务器”都是集成于一个端，此时强制全局使用的机制插件就变得很令人讨厌

2️⃣ 能自定义的常量都**允许自定义**

> 可以省去很多麻烦的事情——例如项目归档后仍有服主跑来找你改内部数据
>
> 不过在编写模板配置时需要注意其**有序性**，太多的常量挤在一个文件中不是很好，例如可恶的Essentials的config.yml，长达千行。

3️⃣ 跟随版本**自动更新**的配置文件

>插件更新时难免会伴随配置文件的修修补补乃至翻天覆地的修改，尝试写一个转换配置版本小函数用不了你多长时间
>
>不然服主们会为每次更新都需要删去文件重新配置而抓狂！

4️⃣ 语言文件的**最大化**，**前缀的分离**

> 很多服主都想让他们的服务器信息统一，因此可修改的前缀是很受欢迎的。
>
> **前缀和后续的语言文字尽量分离**，你只需要多写一行
>
> ```yaml
> prefix: "前缀"
> ```
>
> 但是这可以解决服主修改几百条信息的前缀的麻烦。

5️⃣ **自带注释**的配置

> 没有人想一边翻看教程一边配置！

※ 同时实现自动更新的配置（3️⃣）和自带注解（5️⃣）

> Miao和Karlatemp各写了一篇教程，我忘了链接了......因此直接献上我自己的做法：
>
> https://untiltheend.coding.net/public/UntilTheEnd/UntilTheEnd/git/files/master/src/main/java/ute/Config.java
>
> 其中**autoUpdateConfigs(String name)**即为所求。



### ○ 利用随机数

> 随机数不仅仅提供了随机的数字，我们可以用它干很多事情。

例如某些插件看似**无序不定时**的“温馨提示”，实际上就是**Runnable配合随机数**的效果。

```java
if(Math.random()<0.5){ //有一半的概率执行，谁知道呢？
    //你的代码
}
if(Math.random()<0.0005){ //此处的代码不太可能被执行，但是如果Runnable一直跑下去，谁知道呢？
    //你的代码
}
```



### ○ 拒绝重复

你曾经有没有被这样的代码所困扰：

```java
Inventory inv=Bukkit.createInventory(null,54,"title");
ItemStack item=new ItemStack(Material.IRON_BLOCK);
inv.setItem(0,item);
inv.setItem(2,item);
inv.setItem(5,item);
inv.setItem(9,item);
inv.setItem(10,item);
inv.setItem(16,item);
inv.setItem(18,item);
inv.setItem(30,item);
inv.setItem(45,item);
inv.setItem(46,item);
inv.setItem(48,item);
inv.setItem(52,item);
//OH ! TOO ! MUCH !
```

有些人会把这些set函数压在一行，让代码变得又长又臭，如同酒罐底部的渣滓。

问题是我们需要设置物品的格子ID往往是**无序**的，不能被for循环**一步解决**！

此时引入一个数组，将会使得代码令人神清气爽：

```java
Inventory inv=Bukkit.createInventory(null,54,"title");
ItemStack item=new ItemStack(Material.IRON_BLOCK);
int[] slots=new int[]{0,2,5,9,10,16,18,30,45,46,52};
for(int i=0;i<11;i++)
	inv.setItem(slots[i],item);
```

> 这其实是很普通的一个做法，只是给**无序的slotID们分配有序的下标**，类似于链表但是简单且易懂。



### ○ 插件的Log与Debug信息

一定要把插件的每一个动作都存进**单独的LOG**里！！！！

一定要把插件的每一个动作都存进单独的LOG里！！！！

一定要把插件的每一个动作都存进单独的LOG里！！！！

> 不要问，问就是一年前在网易开服的时候插件出BUG被玩家爆破，又**找不到记录无法惩治**，最后不得已删档。
>
> 谁知道哪些玩家利用了Bug呢？到那时候没有插件会帮你，滚去看千行的服务器Log吧！

另外，这里**存进LOG的信息**和**输出到控制台的信息**又有额外的讲究，那些琐碎的、重复易刷屏的不建议呈现，或是放进**DEBUG模式**再呈现到控制台。

如果你稍微探索下，会发现Logger信息分**Info、Warn、Error**，**严重性递增**，你可以使用它们对信息分级，让信息的**传达性更高**。

```java
JavaPlugin#getLogger().info("普通信息");
JavaPlugin#getLogger().warning("警告");
JavaPlugin#getLogger().severe("错误");

JavaPlugin#getLogger().fine("成功加载");
```



### ○ 提示权限化

服主不希望玩家知道服务器管理层面的内容，你一定也不希望

> 万一这些本该隐藏内容存在Bug，玩家得知后加以利用呢？

因此，我们需要做到提示的**权限化**，以下是一些常用做法：

1️⃣ Help指令 玩家只能看到**有权限执行的指令的帮助**

2️⃣ **插件更新信息只对Op发送**

3️⃣ 根据打开者拥有的权限，**部分隐藏Gui中的按钮或内容**



### ○ 指令TabComplete

美观的指令语法，再配上可爱的Tab功能，是世界上最受人欢迎的东西了。

Bukkit提供了两个实现Tab功能的方式，一个是**TabExecutor**，一个是**TabCompleteEvent**

我更推荐使用前者，毕竟不用注册监听器嘛，还可以把指令一起写了。

只需要**重写onComplete(...)函数**就行了，不过要注意发送者在补全的时候可能**已经拼写了一部分**，因此需要根据**字典序匹配**去生成补全列表：

```java
@Override
public List<String> onTabComplete(CommandSender sender, Command command, String alias, String[] args) {
	String latest = null;
    List<String> list = new ArrayList<>();
    //你的代码，一般根据args的长度、玩家的权限去查找可能会补全的单词，添加进list即可
    if (args.length != 0) {
        latest = args[args.length - 1];
    }
	return filter(latest,list);
}
//字典序筛选功能
//list是原始列表，即在该位置可能出现的所有字符串
//latest是玩家已经输入的一部分，大概是args[length-1]（如果有的话）
private void filter(List<String> list, String latest) {
    if (list.isEmpty() || latest == null)
        return;
    String ll = latest.toLowerCase();
    list.removeIf(k -> !k.toLowerCase().startsWith(ll));
}
```

这个函数和onCommand(...)没啥区别，正常去写就行了，**最后返回可补全的列表即可**。



### ○ 手动寻路算法

如果你看不上原版的寻路算法，可以手写一个。

大概的思路是使用**广度优先搜索**，简称BFS，本篇教程不再赘述，自行上网搜索。

使用**双向BFS**，可以使用**双倍的能耗**，换取**双倍的寻路距离**（若使用普通BFS，双倍路程的计算量将会**指数级增长**）

切记不能使用深度优先搜索DFS或者取直线，不然生物要么找不到路线（此时能耗非常大，详情见DFS的原理），要么撞墙或是掉下悬崖。

另外，注意**控制临界点**，毕竟太远了生物追踪不到嘛。

> BFS不影响性能的范围约为10格。
>
> 如果你的生物的"视野"很远，此时使用BFS势必带来**漫长且冗余**的搜索。我们需要动脑筋，先让生物靠近一点或者使用A*等更高端的**带估值期望的算法**。

BFS的另一个用处就是处理连锁挖矿的问题，若使用DFS，挖掘的轨迹将会是**很长的一条线**（BFS是令人愉快的**正八面体**），以下是一个最多100方块的连锁挖矿：

```java
//最多连锁个数
private static int MAXN=100;
//type - 连锁的方块种类
//root - 从哪里开始挖掘
//player - 挖掘的玩家
public static void link(Material type,Location root,Player player){
    //已经遍历的方块个数
    int cnt=0;
    //把根源加进去
	ArrayList<Location> queue=new ArrayList<>();
    queue.add(root);
    //仍然有方块可以遍历
    while(cnt++<MAXN && (!queue.isEmpty())){
        Location current=queue.get(0);
        //先破坏掉，以免重复，同时省去开visited列表判定是否已经遍历过（因为下面有判断是否是同种方快的if）
        current.getBlock().breakNaturally();
        queue.remove(0);
        //当前方块周围的六个方块
        ArrayList<Location> nearbys=new ArrayList<Location>();
        nearbys.add(current.clone().add(1,0,0));
        nearbys.add(current.clone().add(-1,0,0));
        nearbys.add(current.clone().add(0,1,0));
        nearbys.add(current.clone().add(0,-1,0));
        nearbys.add(current.clone().add(0,0,1));
        nearbys.add(current.clone().add(0,0,-1));
        for(Location nearby:nearbys){
            //如果是同种方块
            if(nearby.getBlock().getType()==type){
                queue.add(nearby);
            }
        }
    }
}
```



### ○ 不同版本的Material切换

高版本和低版本的Material可以说是大相径庭，不过Bukkit给我们留了一条后路：**LEGACY_前缀**

这样我们就可以轻易地转换高低版本的Material，已经写好的代码如下：

```java
import org.bukkit.Bukkit;
import org.bukkit.Material;
import org.bukkit.UnsafeValues;
import org.bukkit.block.Block;
import org.bukkit.inventory.ItemStack;
import org.bukkit.inventory.meta.ItemMeta;

import java.lang.invoke.LambdaMetafactory;
import java.lang.invoke.MethodHandles;
import java.lang.invoke.MethodType;
import java.util.Objects;
import java.util.function.Function;
import java.util.function.Predicate;

@SuppressWarnings({"unchecked", "JavaLangInvokeHandleSignature"})
public class ItemFactory {
    private static final Function<String, Material> valueOf;
    private static final Function<String, Material> getMaterial;
    private static final Function<ItemStack, Material> getType;
    private static final Function<Block, Material> getTypeBlock;
    private static final Function<Material, Material> fromLegacy;
    private static final Predicate<Material> isLegacy;
    public static final boolean use13;
    public static final Material AIR;

    static {
        MethodHandles.Lookup lk = MethodHandles.lookup();
        Predicate<Material> isLeg = v -> false;
        try {
            valueOf = (Function<String, Material>) LambdaMetafactory.metafactory(lk, "apply", MethodType.methodType(Function.class),
                    MethodType.methodType(Object.class, Object.class),
                    lk.findStatic(Material.class, "valueOf", MethodType.methodType(Material.class, String.class)),
                    MethodType.methodType(Material.class, String.class)).getTarget().invoke();
            getMaterial = (Function<String, Material>) LambdaMetafactory.metafactory(lk, "apply", MethodType.methodType(Function.class),
                    MethodType.methodType(Object.class, Object.class),
                    lk.findStatic(Material.class, "getMaterial", MethodType.methodType(Material.class, String.class)),
                    MethodType.methodType(Material.class, String.class)).getTarget().invoke();
            getType = (Function<ItemStack, Material>) LambdaMetafactory.metafactory(lk, "apply", MethodType.methodType(Function.class),
                    MethodType.methodType(Object.class, Object.class),
                    lk.findVirtual(ItemStack.class, "getType", MethodType.methodType(Material.class)),
                    MethodType.methodType(Material.class, ItemStack.class)).getTarget().invoke();
            getTypeBlock = (Function<Block, Material>) LambdaMetafactory.metafactory(lk, "apply", MethodType.methodType(Function.class),
                    MethodType.methodType(Object.class, Object.class),
                    lk.findVirtual(Block.class, "getType", MethodType.methodType(Material.class)),
                    MethodType.methodType(Material.class, Block.class)).getTarget().invoke();
            Function<Material, Material> f;
            boolean u13 = true;
            try {
                f = (Function<Material, Material>) LambdaMetafactory.metafactory(lk, "apply", MethodType.methodType(Function.class, UnsafeValues.class),
                        MethodType.methodType(Object.class, Object.class),
                        lk.findVirtual(UnsafeValues.class, "fromLegacy", MethodType.methodType(Material.class, Material.class)),
                        MethodType.methodType(Material.class, Material.class)).getTarget().invoke(Bukkit.getUnsafe());
            } catch (Throwable ignore) {
                f = Function.identity();
                u13 = false;
            }
            use13 = u13;
            fromLegacy = f;
            try {
                isLeg = (Predicate<Material>) LambdaMetafactory.metafactory(lk, "test", MethodType.methodType(Predicate.class),
                        MethodType.methodType(boolean.class, Object.class),
                        lk.findVirtual(Material.class, "isLegacy", MethodType.methodType(boolean.class)),
                        MethodType.methodType(boolean.class, Material.class)).getTarget().invoke();
            } catch (Throwable exception) {
                if (u13) {
                    throw new ExceptionInInitializerError(exception);
                }
            }
        } catch (Throwable throwable) {
            throw new ExceptionInInitializerError(throwable);
        }
        isLegacy = isLeg;
        AIR = valueOf("AIR");
    }

    public static Material getType(ItemStack stack) {
        return getType.apply(stack);
    }

    public static Material getMaterial(String material) {
        return getMaterial.apply(material);
    }

    public static Material getType(Block block) {
        return getTypeBlock.apply(block);
    }

    public static boolean isLegacy(Material material) {
        return isLegacy.test(material);
    }

    public static Material valueOf(String name) {
        try {
            return valueOf.apply(name);
        } catch (Throwable any) {
            // 低于 1.13 版本的时候没必要再去搜索 LEGACY
            if (!use13) throw any;
        }
        Material result = fromLegacy.apply(valueOf.apply("LEGACY_" + name));
        if (result == AIR) {
            if (!name.equals("AIR")) {
                throw new IllegalArgumentException("No enum constant Material." + name +
                        " (constant founded but result of flatting is AIR)");
            }
        }
        return result;
    }

    public static Material load(String... names) {
        for (String name : names) {
            try {
                return valueOf(name);
            } catch (Throwable ignore) {
            }
        }
        throw new RuntimeException(String.join(", ", names));
    }

    public static Material fromLegacy(Material material) {
        if (isLegacy(material)) return fromLegacy.apply(material);
        return material;
    }

    public static boolean isSame(ItemStack source, ItemStack check) {
        if (source == check) return true;
        if (source == null || check == null) return false;
        if (use13) {
            if (source.getAmount() != check.getAmount()) return false;
            if (source.hasItemMeta() != check.hasItemMeta()) return false;
            if (fromLegacy(getType(source)) != fromLegacy(getType(check))) return false;
            final ItemMeta i1 = source.getItemMeta();
            final ItemMeta i2 = check.getItemMeta();
            if (i1 == null && i2 == null) return true;
            if (i1 == null || i2 == null) return false;
            if (!Objects.equals(i1.getDisplayName(), i2.getDisplayName())) return false;
            if (!Objects.equals(i1.getLore(), i2.getLore())) return false;
            if (source.getDurability() != check.getDurability()) return false;
            return Objects.equals(i1.getEnchants(), i2.getEnchants());
        } else return source.equals(check);
    }

    public static String toString(Object material) {
        return String.valueOf(material);
    }
}
```

你只需要使用**ItemFactory.fromLegacy(String typeName)**获取Material即可，这里的typeName可以是高版本也可以是低版本。

需要注意的是，如果你在高版本中，对使用以上方法生成的**旧的ItemStack**进行**getType()**操作，得到的**Material是高版本**的，而此时两个ItemStack进行比对时就不会相等。建议自己**动手写equals函数**。

> 另外，上面的代码让你无需在兼容不同版本时切换库，或者疯狂使用ItemFactory.fromLegacy(String typeName)，例如在1.12的构建环境下直接调用**Material.WOOL**，插件放到1.13+中**不会报错**，上面的代码会自动做后续处理。



### ○ 各种Inventory的SlotID大表（1.12）

以下是我"打印"的SlotID大表图片，欢迎直接伸手：

> **格子ID=白色玻璃的个数-1**
>
> **格子ID=灰色玻璃个数+63**

![](https://gitee.com/HamsterYDS/MyImages/raw/master/mcPluginImg/20210612152126.png)

![BEACON](https://gitee.com/HamsterYDS/MyImages/raw/master/mcPluginImg/20210612152124.png)

![BREWING](https://gitee.com/HamsterYDS/MyImages/raw/master/mcPluginImg/20210612152122.png)

![CHEST](https://gitee.com/HamsterYDS/MyImages/raw/master/mcPluginImg/20210612152120.png)

![CREATIVE](https://gitee.com/HamsterYDS/MyImages/raw/master/mcPluginImg/20210612152118.png)

![DISPENSER](https://gitee.com/HamsterYDS/MyImages/raw/master/mcPluginImg/20210612152116.png)

![DROPPER](https://gitee.com/HamsterYDS/MyImages/raw/master/mcPluginImg/20210612152113.png)

![ENCHANTING](https://gitee.com/HamsterYDS/MyImages/raw/master/mcPluginImg/20210612152109.png)

![ENDERCHEST](https://gitee.com/HamsterYDS/MyImages/raw/master/mcPluginImg/20210612152051.png)

![FURNACE](https://gitee.com/HamsterYDS/MyImages/raw/master/mcPluginImg/20210612152048.png)

![HOPPER](https://gitee.com/HamsterYDS/MyImages/raw/master/mcPluginImg/20210612152046.png)

![PLAYER](https://gitee.com/HamsterYDS/MyImages/raw/master/mcPluginImg/20210612152043.png)

![SHULKERBOX](https://gitee.com/HamsterYDS/MyImages/raw/master/mcPluginImg/20210612152037.png)

![WORKBENCH](https://gitee.com/HamsterYDS/MyImages/raw/master/mcPluginImg/20210612152033.png)



### ○ Inventory安全性

1️⃣ 给予物品

测试可知在满背包时直接Inventory#addItem(ItemStack item)是**无效**的，它会返回一个Map，其中包含**给予失败的物品**，通过这个Map我们可以进行后续的处理。

如果你懒的话，可以直接模拟**“背包溢出”**——在玩家位置处**掉落给予失败的物品**，然而这难免会产生玩家和玩家或玩家和服主之间的纠纷。

> 万一他们没有得到该得到的，胡闹一通呢？
>
> 万一他们假装没有得到，胡闹一通呢？

这些问题往往困扰服主，此时就是开发者做善事的时候了：

做一个类似于**物品中转站**的功能，给予失败的物品都放入此中转站中，如果过了一段时间玩家**仍未领取，则自动清理**。

2️⃣ 可能存在的卡Bug方式

取消你的Gui的InventoryClickEvent是远远不够的，因为玩家很有可能点击自己的背包内的物品，**拖拽（Drag）到你的Gui中**。

如果没有上传物品等功能，**仅仅需要点击的Gui**，建议开一个Set存储打开特殊Gui的玩家们并禁止**一切**该Set中的玩家的Gui操作

在玩家打开Gui时就将其记录到Set中，在玩家关闭Gui时就将其从Set中移去。



### ○ ItemMeta的子类

ItemMeta的子类有很多，在实际使用时，**将ItemMeta转换类型后操作，再set回ItemStack**即可。以下是两个例子：

1️⃣ 自定义头颅 SkullMeta

获取某个玩家的头颅：

```java
ItemStack item=new ItemStack(Material.SKULL);
SkullMeta meta=(SkullMeta) item.getItemMeta();
meta.setOwningPlayer(Bukkit.getOfflinePlayer("玩家名"));
item.setItemMeta(meta);
```

如果你想使用自己的头颅贴图，可以去了解**CSCoreLib**

2️⃣ 刷怪蛋 SpawnEggMeta

获取某种生物对应的刷怪蛋：

```java
ItemStack item = new ItemStack(Material.MONSTER_EGG);
SpawnEggMeta meta = (SpawnEggMeta) item.getItemMeta();
meta.setSpawnedType(EntityType.COW/*实体种类*/);
item.setItemMeta(meta);
```



### ○ 真实伤害与免疫后伤害

当你使用**Damagable#damage(double amount)**时，会发现参数为**真实伤害**，绕过了原版盔甲、免疫等的计算。

如果你要进行原版伤害计算系统复现，以下对你有所帮助：

1️⃣ 盔甲及附魔

※ 盔甲值与韧性

![ArmorDamageFormula Simplified.svg](https://gitee.com/HamsterYDS/MyImages/raw/master/mcPluginImg/20210612154404.png)

![Damage dealt v3 Simplified.png](https://gitee.com/HamsterYDS/MyImages/raw/master/mcPluginImg/20210612154424.png)

![Damage reduction v3 Simplified.png](https://gitee.com/HamsterYDS/MyImages/raw/master/mcPluginImg/20210612154431.png)

※ 盔甲的附魔：**附魔计算后伤害=计算前伤害* ( 1 - EPF综合 / 25 )**

|    魔咒    |                        能够减少的伤害                        | 伤害修正权重 | EPF 等级I | EPF 等级II | EPF 等级III | EPF 等级IV |
| :--------: | :----------------------------------------------------------: | :----------: | :-------: | :--------: | :---------: | :--------: |
|    保护    |                             全部                             |      1       |     1     |     2      |      3      |     4      |
|  火焰保护  | [火](https://minecraft.fandom.com/zh/wiki/火)、[熔岩](https://minecraft.fandom.com/zh/wiki/熔岩)、[岩浆块](https://minecraft.fandom.com/zh/wiki/岩浆块)和[烈焰人](https://minecraft.fandom.com/zh/wiki/烈焰人)火球 |      2       |     2     |     4      |      6      |     8      |
|  爆炸保护  |      [爆炸](https://minecraft.fandom.com/zh/wiki/爆炸)       |      2       |     2     |     4      |      6      |     8      |
| 弹射物保护 | [箭](https://minecraft.fandom.com/zh/wiki/弓)、[恶魂](https://minecraft.fandom.com/zh/wiki/恶魂)和[烈焰人](https://minecraft.fandom.com/zh/wiki/烈焰人)火球 |      2       |     2     |     4      |      6      |     8      |
|  摔落保护  | 掉落伤害（包括[末影珍珠](https://minecraft.fandom.com/zh/wiki/末影珍珠)） |      3       |     3     |     6      |      9      |     12     |

2️⃣ 药水效果

抗性提升效果：**计算后伤害=计算前伤害(1-效果等级0.2)**

伤害吸收效果：**不需要额外判断，经测试，damage(double)会优先扣除黄色心。**

3️⃣ 伤害的原因

原版伤害：

| 伤害种类 | 原始伤害计算公式或影响因素 | 免疫后伤害计算                                               |
| -------- | -------------------------- | ------------------------------------------------------------ |
| 摔落伤害 | (H-3) ❤                    | 盔甲、摔落保护、保护、抗性提升                               |
| 格斗伤害 | 工具、攻击者、药水效果     | 盔甲、保护、弹射物保护等                                     |
| 火焰伤害 | 岩浆每秒6❤，灼烧每秒1❤     | 盔甲、火焰保护、保护、抗性提升、抗火（完全免疫）             |
| 魔法伤害 | 药水种类、等级             | 抗性提升                                                     |
| 爆炸伤害 | 取决于爆炸距离、爆炸强度   | 盔甲、爆炸保护、保护、抗性提升                               |
| 窒息伤害 | 每秒1❤                     | 生存模式下无法被减免，可以通过水下呼吸增长水造成的窒息伤害计算周期 |
| 虚空伤害 | 每秒2❤                     | 任何模式下无法被减免                                         |
| 饥饿伤害 | 每秒0.2❤                   | 饱和                                                         |

至于你自己杜撰的伤害，无非分为两种：

| 伤害种类 | 免疫后伤害计算                                             |
| -------- | ---------------------------------------------------------- |
| 物理伤害 | 玩家拿着剑（1.8-）或盾牌（1.9+）格挡                       |
| 魔法伤害 | 忽略盔甲的阻挡效果、玩家拿着鳕鱼的情况（仅个人看法[doge]） |

> 当然，你的插件创造的伤害可能还有别的分类，**这取决于你**。

综上，从原始伤害计算免疫后伤害代码如下（这里不展示原始伤害计算过程）：

//TODO

> 另外，你的damage可能会**导致玩家死亡**，然而“某某某死了”是很难看的死亡信息，而且起不到公示的效果，因此你可以考虑**监听死亡事件**，为不同的死亡**输出不同的信息**。



### ○ 手动判等ItemStack

有一些物品看上去一样，但是使用equals判断时**并不相等**，这可能是因为：

1️⃣ Name不一样（例如RPGItem会改掉物品名方便其判定，在其前面增加无用的稀奇古怪的颜色代码）。

2️⃣ Material不一样（例如上文提到的1.13+中LEGACY_WOOL和WHITE_WOOL）。

3️⃣ 其他，例如NBT和Lore等，不过这些一般不会出错。

此时你需要手动判断，把**有关字符串的处理掉颜色代码**，把**LEGACY_的Material变成其对应值**再比对。



### ○ 光照机制计算

原版的光照系统这里就不全面介绍了，这里只提几个注意点：

1️⃣ 天空光照不随白天黑夜交替而改变，**永远都是15**。

2️⃣ 完全透明的方块完全透光，半透明的方块不透天空的光，不透明的方块不透光。

3️⃣ **天气对天空光照的影响**

|                          时间+天气                           | 实际天空光照 |
| :----------------------------------------------------------: | :----------: |
|                         中午，晴天时                         |      15      |
| 中午，[降雨](https://minecraft.fandom.com/zh/wiki/降雨)或[降雪](https://minecraft.fandom.com/zh/wiki/降雪)时 |      12      |
|  中午，[雷暴](https://minecraft.fandom.com/zh/wiki/雷暴)时   |      10      |
|                         午夜，晴天时                         |      4       |

这里提一个很新奇的玩意——**不存在的光源**：**LightAPI——https://www.mcbbs.net/thread-1016938-1-1.html**



### ○ 序列化模板

1️⃣ **实体序列化（通过NBT）**： From 果粒姐姐

实体转换成NBT：

![](https://gitee.com/HamsterYDS/MyImages/raw/master/mcPluginImg/20210612163446.png)

NBT转换成实体：

![](https://gitee.com/HamsterYDS/MyImages/raw/master/mcPluginImg/20210612163651.png)

2️⃣ **坐标/区块序列化模板（Loc3D.locToStr(Location loc)和Loc3D.strToLoc(String string)）**：

```java
import com.google.common.annotations.Beta;
import com.google.common.io.ByteArrayDataInput;
import com.google.common.io.ByteArrayDataOutput;
import com.google.common.io.ByteStreams;
import org.bukkit.Bukkit;
import org.bukkit.Location;
import org.bukkit.World;

import java.io.Serializable;
import java.util.Base64;
import java.util.NoSuchElementException;
import java.util.Objects;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public class Loc3D implements Serializable {
    private static final long serialVersionUID = 148757128437582932L;
    public final int x, y, z;
    public final String world;

    public Loc3D(String world, int x, int y, int z) {
        this.x = x;
        this.y = y;
        this.z = z;
        this.world = world;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;

        Loc3D loc3D = (Loc3D) o;

        if (x != loc3D.x) return false;
        if (y != loc3D.y) return false;
        if (z != loc3D.z) return false;
        return Objects.equals(world, loc3D.world);
    }

    @Override
    public int hashCode() {
        int result = x;
        result = 31 * result + y;
        result = 31 * result + z;
        result = 31 * result + (world != null ? world.hashCode() : 0);
        return result;
    }

    public Location toLocation() {
        World world = this.world == null ? null : Bukkit.getWorld(this.world);
        return new Location(world, x, y, z);
    }

    public Location toUsableLocation() {
        Location loc = toLocation();
        if (loc.getWorld() == null) {
            throw new NoSuchElementException("No world `" + world + "` found.");
        }
        return loc;
    }

    public static Loc3D from(Location location) {
        World world = location.getWorld();
        String worldName = world == null ? null : world.getName();
        return new Loc3D(worldName, location.getBlockX(), location.getBlockY(), location.getBlockZ());
    }
    @SuppressWarnings("UnstableApiUsage")
    private static Loc3D binToLoc(byte[] data) {
        final ByteArrayDataInput input = ByteStreams.newDataInput(data);
        String worldName = input.readUTF();
        return new Loc3D(
                worldName,
                input.readInt(),
                input.readInt(),
                input.readInt()
        );
    }

    @Beta
    public static final Pattern loc3d = Pattern.compile(
            // World-X-Y-Z
            "(^.*)?((-|)[0-9]+)-((-|)[0-9]+)-((-|)[0-9]+)$"
    );

    public static Loc3D strToLoc3d(String location) {
        try {
            return strToLoc0(location);
        } catch (Throwable any) {
            any.addSuppressed(new Exception("Try parsing `" + location + "`"));
            throw any;
        }
    }

    public static Location strToLoc(String location) {
        final Loc3D loc3d = strToLoc3d(location);
        if (loc3d == null) return null;
        try {
            return loc3d.toUsableLocation();
        } catch (Throwable any) {
            any.addSuppressed(new Exception("Try parsing `" + location + "`"));
            throw any;
        }
    }

    public static Loc3D strToLoc0(String location) {
        base64:
        {
            byte[] base64;
            try {
                base64 = Base64.getDecoder().decode(location);
            } catch (Exception ignored) {
                break base64;
            }
            return binToLoc(base64);
        }
        final Matcher matcher = loc3d.matcher(location);
        if (!matcher.find()) {
            return null;
        }
        // region Legacy

        // region 正则匹配信息
        // 1: World
        // 2: X
        // 4: Y
        // 6: Z
        // endregion
        String world = matcher.group(1);
        String x = matcher.group(2);
        String y = matcher.group(4);
        String z = matcher.group(6);
        if (world.isEmpty()) throw new ArrayIndexOutOfBoundsException("Empty world name");
        if (x.charAt(0) != '-') { // 应该永远都是 true?
            int cut = world.length();
            while (cut > 1) {
                int pre = cut - 1;
                char prev = world.charAt(pre);
                if (prev == '-') {
                    cut = pre;
                    break;
                } else {
                    if (prev >= '0' && prev <= '9') {
                        cut = pre;
                    } else break;
                }
            }
            String fullWorld = world;
            world = fullWorld.substring(0, cut);
            x = fullWorld.substring(cut) + x;
        }
        return new Loc3D(
                world,
                Integer.parseInt(x),
                Integer.parseInt(y),
                Integer.parseInt(z)
        );
        // endregion
    }

    @SuppressWarnings("UnstableApiUsage")
    public static String locToStr(Loc3D loc) {
        final ByteArrayDataOutput output = ByteStreams.newDataOutput();
        output.writeUTF(loc.world);
        output.writeInt(loc.x);
        output.writeInt(loc.y);
        output.writeInt(loc.z);
        return Base64.getEncoder().encodeToString(output.toByteArray());
    }

    public static String locToStr(Location loc) {
        return locToStr(Loc3D.from(loc));
    }
}
```



//以下内容等待更新！

### ○ 正版登陆判定

//TODO



### ○ NBT标签操作

attribute管理，最大血量，nbt标签



### ○ 大型方块与组合碰撞箱

大型方块的方向、碰撞箱拆解



### ○ 隐去效果粒子，改变效果颜色

药水效果的粒子隐形

光灵效果的颜色



### ○ 实体的速度

玩家速度更改与概论



### ○ 方块集合模板

透明方块集合、方块类型集合

------

## 三、功能|思路（From FTS）

### ○ 自定义原版升级经验

待讲解

```java
import fts.FunctionalToolSet;
import fts.spi.ResourceUtils;
import org.bukkit.Bukkit;
import org.bukkit.configuration.file.YamlConfiguration;
import org.bukkit.entity.Player;
import org.bukkit.event.EventHandler;
import org.bukkit.event.Listener;
import org.bukkit.event.player.PlayerExpChangeEvent;
import org.bukkit.scheduler.BukkitRunnable;

import java.io.File;
import java.util.HashMap;

public class CustomLevelExp implements Listener {
    private static final HashMap<Integer, Integer> expNeedToUpgrade = new HashMap<>();

    public static void initialize(FunctionalToolSet plugin) {
        ResourceUtils.autoUpdateConfigs("customexp.yml");
        File file = new File(plugin.getDataFolder(), "customexp.yml");
        YamlConfiguration yaml = YamlConfiguration.loadConfiguration(file);
        if (!yaml.getBoolean("enable")) {
            return;
        }

        for (String path : yaml.getKeys(false)) {
            if (path.equalsIgnoreCase("enable")) {
                continue;
            }
            int level = Integer.parseInt(path) - 1;
            int exp = yaml.getInt(path);
            expNeedToUpgrade.put(level, exp);
        }

        Bukkit.getPluginManager().registerEvents(new CustomLevelExp(), plugin);
    }

    private static int getExpToLevel(int level) {
        if (level <= 15) {
            return 2 * level + 7;
        } else if (level <= 30) {
            return 5 * level - 38;
        } else {
            return 9 * level - 158;
        }
    }

    @EventHandler
    public void onExp(PlayerExpChangeEvent event) {
        Player player = event.getPlayer();
        int exp = event.getAmount();
        int level = player.getLevel();
        if (expNeedToUpgrade.containsKey(level)) {
            event.setAmount(0);
            float current = player.getExp() * expNeedToUpgrade.get(level);
            new BukkitRunnable() {
                @Override
                public void run() {
                    if (current + exp >= expNeedToUpgrade.get(level)) {
                        player.setLevel(player.getLevel() + 1);
                        if (expNeedToUpgrade.containsKey(level + 1)) {
                            player.setExp((current + exp - expNeedToUpgrade.get(level)) / expNeedToUpgrade.get(level));
                        } else {
                            player.setExp((current + exp - expNeedToUpgrade.get(level)) / getExpToLevel(level + 1));
                        }
                        player.setLevel(level + 1);
                    } else if (current + exp < 0) {
                        player.setLevel(player.getLevel() - 1);
                        if (expNeedToUpgrade.containsKey(level - 1)) {
                            player.setExp((expNeedToUpgrade.get(level - 1) + current + exp) / expNeedToUpgrade.get(level - 1));
                        } else {
                            player.setExp((getExpToLevel(level - 1) + current + exp) / getExpToLevel(level - 1));
                        }
                        player.setLevel(level - 1);
                    } else {
                        player.setExp((current + exp) / expNeedToUpgrade.get(level));
                        player.setLevel(level);
                    }
                }
            }.runTaskLater(FunctionalToolSet.getInstance(), 1L);
        }
    }
}

```

------

## 四、支持作者

### ● 回复评分帖子

> 评人气不会扣自己的哦~

### ● 加入QQ交流群

> UntilTheEnd|FunctionalToolSet|官方：1051331429

### ● 打赏

> 爱发电：http://afdian.net/@HamsterYDS

------

### 来自群组：Server CT