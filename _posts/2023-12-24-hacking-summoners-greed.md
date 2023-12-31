---
layout: post
title: Hacking a mobile game and making a save editor
description: I looked into the save format of the mobile game 'Summoners Greed', how it works, how to cheat and how to automate it!
published: true
---
I looked into the save format of the mobile game 'Summoners Greed', how it works, how to cheat and how to automate it!

## Part 1: Where's the save file?
When initially looking at this game, I had no internet connection and was a little bit bored (avoiding school work). So after looking at the game I was wondering if there was a way to cheat.

I connected my phone to my laptop and had a look through the files. My first look was at the `/Android/data` folder.

After looking for a little while I noticed `com.pixio.google.mtd`, 'Pixio' being the name of the developers.

Inside was the following directory:

```
cache/
files/
no_backup/
```

`files` prooved to be the most interesting, after seeing `user.gamedata` I knew I'd found my file!

Interestingly there was also `app.gamedata`, which we'll come back to...

## How can we cheat?
An initial look at `user.gamedata` showed it was an SQLite3 file.
```
$ file user.gamedata
user.gamedata: SQLite 3.x database, last written using SQLite version 3028000, file counter 2557, database pages 57, cookie 0x3c, schema 4, UTF-8, version-valid-for 2557
```

Opening the file with `sqlite3 ./user.gamedata` allowed us to interact with the save data.

Running `.tables` listed the tables inside the database.
```
sqlite> .tables
android_metadata           user_pvp_store_product
user_achievement           user_server_mail
user_bonus                 user_skill_tree_node
user_charm                 user_stats
user_currency              user_store_product_bundle
user_data                  user_team
user_enemy                 user_tower
user_event                 user_tower_group
user_friend_invite         user_tower_slot
user_map                   user_tower_unlock
user_obstacle              user_tutorial
user_player_skill          user_wave
user_pvp_season            user_weekly_achievement
user_pvp_season_player
```

Particularly of interest was `user_currency`, `user_skill_tree_node`, `user_charm` and `user_tower`.

```
sqlite> .schema user_currency
"user_id" integer primary key autoincrement not null ,
"current_map_id" integer ,
"user_coin" bigint ,
"user_gem" bigint ,
"user_orb" bigint ,
"user_power_stone" bigint ,
"user_evolution_stone" bigint ,
"user_evolution_stone_new" bigint ,
"current_wave" integer ,
"action_type" varchar ,
"next_bonusman_wave" integer ,
"retry_count" integer ,
"free_retry_remain_time" integer ,
"inited" integer ,
"launch_time" datetime ,
"launch_version" varchar ,
"game_time" float ,
"sign_out_time" datetime ,
"last_fb_share" datetime ,
"element_normal_max_damage" bigint ,
"element_hard_max_damage" bigint ,
"total_damage" bigint ,
"invite_reward_count" integer ,
"evo_gem_loot" integer ,
"master_challenge_resul" varchar ,
"user_master_coin" bigint ,
"user_energy" integer ,
"energy_remain_time" integer ,
"energy_last_update_time" float ,
"offline_coin" bigint ,
"sign_out_server_time" datetime ,
"client_offlinecoin_stop" integer ,
"clicked_link_gem_media" integer ,
"user_evolution_ice_stone" bigint ,
"user_evolution_fire_stone" bigint ,
"user_mythical_stone" bigint ,
"user_clicked_link_reward" integer ,
"return_pack_on" integer ,
"user_evolution_lightning_stone" bigint , "user_charm_stone" bigint, "user_seq_json" varchar, "user_fish_stone" bigint, "user_fish_power_stone" bigint, "user_pirate_intro_state" integer, "user_super_mythical_stone" bigint, "user_double_speed_ads_timer" float, "daily_ads_double_speed_count" integer, "last_double_speed_watch_video_time" bigint, "no_ads" integer, "user_pvp_upgrade_coin" bigint, "user_pvp_shop_coin" bigint, "user_common_star" bigint, "user_epic_star" bigint, "user_rare_star" bigint, "user_Legendary_star" bigint, "user_ultra_star" bigint, "user_mythical_star" bigint, "claim_pvp_reward" integer, "user_recycle_token" bigint, "user_pvp_vip_end" bigint);
```

Looking through, we have a few 'currency' enteries from the game.

- Coins: `user_coin`
- Gems: `user_gem`
- Green Orbs: `user_orb`
- and so on with other assorted currencies...

Let's try update our coins for the game.

```
sqlite> SELECT user_coin FROM user_data;
1000
sqlite> UPDATE user_data SET user_coin=99999999999999999999999;
sqlite> SELECT user_coin FROM user_data;
1.0e+23
```

Perfect, works a treat in-game.

### The Skill Tree

The game has a 'skill tree', which gives permanent upgrades to towers.
Looking at the tables the `user_skill_tree_node` jumps out at me.

```
sqlite> .schema user_skill_tree_node
CREATE TABLE IF NOT EXISTS "user_skill_tree_node"(
"skill_tree_node_id" integer primary key not null ,
"level" integer );
sqlite>
sqlite> SELECT * FROM user_skill_tree_node;
101|5
102|5
103|5
...
496|5
497|5
498|5
```

The format is `skill_tree_node_id|level`, so if we set all of the id's to level 5, we should auto-complete the skill tree!

### Charms
Charms are small equippables for towers that improve stats for certain types.

```
sqlite> .schema user_charm
CREATE TABLE IF NOT EXISTS "user_charm"(
"user_charm_id" integer primary key autoincrement not null ,
"rarity" integer ,
"upgrade" integer ,
"main_effect_id" integer ,
"special_effect1_id" integer ,
"special_effect2_id" integer ,
"special_effect3_id" integer ,
"special_effect1_amount" float ,
"special_effect2_amount" float ,
"special_effect3_amount" float ,
"ref_mythical_id" integer ,
"equiped_tower_groupId" integer );
```

There are 15 unique types for charms.
- Attack (1)
- Fire Atk (2)
- Ice Atk (3)
- Lightning Atk (4)
- Cosmic Atk (5)
- Elemental Atk (6)
- Crit Dmg (7)
- Attack Spd (8)
- Crit Rate (9)
- Skill CD (10)
- Attack Buff (11)
- Stun Time (12)
- Freeze Time (13)
- Coin (14)
- Power Stone (15)

There are 4 slots for our types, `main_effect_id` and `special_effect1_id` to `special_effect3_id`

Creating all combinations with no duplicates (only one charm can have `1,2,3,4`, `4,2,3,1` and `3,4,2,1` cannot exit) give us `1471` combos.

Our schema has some more interesting values: `special_effect1_amount` to `special_effect3_amount`
By customising these, we can make our charms unfairly strong with a super-high modifier. Awesome!

### Towers
Towers are deployables that deal damage with abilities.

```
sqlite> .schema user_tower
CREATE TABLE IF NOT EXISTS "user_tower"(
"tower_id" integer primary key not null ,
"level" integer ,
"gacha_level" integer ,
"piece_count" integer ,
"is_new" integer ,
"ultra_currency" integer ,
"extra_level" integer ,
"master_level" integer , "season_level" integer, "season_ref" varchar);
```

The `tower_id` dictates which tower, editing the `level` and `extra_level` gives us the maximum level for the tower. Modifying the `gacha_level` allows us to get all the stars on the tower.

Some towers seem to be capped at level 400, some at level 500, all of this is defined in the `app.gamedata` file.

## How can we do it automatically..?

While manually changing values is great, we can make an automated save editor that uses the latest `app.gamedata` and `user.gamedata` to automatically create our values.

