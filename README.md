const LEVEL_COUNT = 20;

class Boot extends Phaser.Scene {
  constructor(){ super('Boot'); }
  preload(){
    // no external assets; we'll use graphics generated at runtime
  }
  create(){
    this.scene.start('Game', {level: 1});
  }
}

class Game extends Phaser.Scene {
  constructor(){ super('Game'); }
  init(data){
    this.level = data.level || 1;
  }
  create(){
    // world bounds and camera
    this.cameras.main.setBackgroundColor('#0b0b1a');
    this.physics.world.setBounds(0, 0, 3200, 600);
    this.cameras.main.setBounds(0,0,3200,600);

    // level data (simple platform arrays) - procedurally create complexity by level
    this.createLevel(this.level);

    // player
    this.player = this.createPlayer(100, 450);
    this.cameras.main.startFollow(this.player, true, 0.08, 0.08);

    // input
    this.cursors = this.input.keyboard.addKeys({
      left: Phaser.Input.Keyboard.KeyCodes.LEFT,
      right: Phaser.Input.Keyboard.KeyCodes.RIGHT,
      up: Phaser.Input.Keyboard.KeyCodes.UP,
      w: Phaser.Input.Keyboard.KeyCodes.W,
      attack: Phaser.Input.Keyboard.KeyCodes.J,
      grapple: Phaser.Input.Keyboard.KeyCodes.K,
      restart: Phaser.Input.Keyboard.KeyCodes.R
    });

    // hud
    this.levelText = this.add.text(16,16, 'Level: ' + this.level, {font:'20px Arial', fill:'#fff'}).setScrollFactor(0);
    this.add.text(16,40, 'Enemies defeated: 0', {font:'14px Arial', fill:'#aaa'}).setScrollFactor(0).setName('scoreText');

    // collisions
    this.physics.add.collider(this.player, this.platforms);
    this.physics.add.collider(this.enemies, this.platforms);
    this.physics.add.collider(this.enemies, this.enemies);
    this.physics.add.overlap(this.player, this.enemies, this.playerHit, null, this);
    this.physics.add.overlap(this.player, this.goal, this.reachGoal, null, this);
  }

  createLevel(levelNum){
    // destroy previous groups if exist
    if(this.platforms) this.platforms.clear(true,true);
    if(this.enemies) this.enemies.clear(true,true);
    if(this.goal) this.goal.destroy();

    this.platforms = this.physics.add.staticGroup();
    this.enemies = this.physics.add.group();

    // width scales with level
    const width = 800 + (levelNum-1)*120;
    this.physics.world.setBounds(0,0,Math.max(width,960),600);
    // ground
    this.platforms.create(width/2, 580, null).setScale(width/960,1).refreshBody().setTint(0x222222);

    // create platforms increasing in complexity
    const platformCount = Math.min(6 + Math.floor(levelNum/2), 18);
    for(let i=0;i<platformCount;i++){
      const px = 150 + i * (Math.max(120, Math.floor(width/platformCount)));
      const py = 400 - ((i%4)*60) - Math.min( (levelNum-1)*3, 200 );
      const p = this.add.rectangle(px, py, 180, 20, 0x333344);
      this.physics.add.existing(p, true);
      this.platforms.add(p);
      // chance to add small enemy on platform
      if(Math.random() < Math.min(0.4, 0.1 + levelNum*0.03)){
        const e = this.createEnemy(px, py-40, levelNum);
        this.enemies.add(e);
      }
    }

    // hazards for higher levels
    for(let i=0;i<Math.floor(levelNum/5);i++){
      const hx = 200 + i*300;
      const hazard = this.add.rectangle(hx, 560, 80, 40, 0x550000);
      this.physics.add.existing(hazard, true);
      // simple visual only - overlaps will check by coordinate
    }

    // place a boss or named villain as the goal at far right
    const goalX = Math.max(width - 80, 700);
    this.goal = this.physics.add.staticImage(goalX, 500, null);
    this.goal.setSize(32, 120);
    this.goal.villain = this.getVillainForLevel(levelNum);
    // small indicator
    this.add.text(goalX-60, 450, this.goal.villain.name, {font:'16px Arial', fill:'#ff6'}).setScrollFactor(0).setDepth(2);
  }

  createPlayer(x,y){
    const p = this.physics.add.sprite(x,y, null);
    p.setSize(28,48);
    p.setBounce(0.1);
    p.setCollideWorldBounds(true);
    p.health = 5;
    // draw using graphics texture
    const g = this.add.graphics();
    g.fillStyle(0x000000,1).fillRect(0,0,32,48);
    g.fillStyle(0x222222,1).fillRect(2,2,28,44);
    g.generateTexture('playerTex',32,48);
    g.destroy();
    p.setTexture('playerTex');
    return p;
  }

  createEnemy(x,y,level){
    const e = this.physics.add.sprite(x,y, null);
    e.setSize(24,36);
    e.setCollideWorldBounds(true);
    e.setBounce(0.2);
    e.health = 1 + Math.floor(level/5);
    e.speed = 30 + Math.min(60, level*4);
    // texture
    const g = this.add.graphics();
    g.fillStyle(0x444444,1).fillRect(0,0,24,36);
    g.generateTexture('enemyTex'+Math.random(),24,36);
    g.destroy();
    e.setTexture(Object.keys(this.textures.list).slice(-1)[0]);
    // simple patrol tween
    const range = 80 + (level*2);
    this.tweens.add({
      targets: e,
      x: e.x + Phaser.Math.Between(-range, range),
      duration: 2000 + Phaser.Math.Between(0,1000),
      yoyo: true,
      repeat: -1,
      ease: 'Sine.easeInOut'
    });
    return e;
  }

  getVillainForLevel(n){
    const villains = [
      {name:'Thugs'}, {name:'Penguin'}, {name:'Riddler'}, {name:'Killer Croc'},
      {name:'Scarecrow'}, {name:'Mr. Freeze'}, {name:'Catwoman'}, {name:'Firefly'},
      {name:'Black Mask'}, {name:'Two-Face'}, {name:'Clayface'}, {name:'Poison Ivy'},
      {name:'Killer Croc (2)'}, {name:'Deadshot'}, {name:'Mad Hatter'}, {name:'Harley Quinn'},
      {name:'Bane'}, {name:"Ra's al Ghul"}, {name:'Arkham Gauntlet'}, {name:'Joker'}
    ];
    return villains[Math.max(0, Math.min(villains.length-1, n-1))];
  }

  playerHit(player, enemy){
    // simple collision: if player is above enemy, kill enemy; else take damage
    if(player.body.velocity.y > 50 && player.y < enemy.y){
      enemy.destroy();
      const st = this.children.getByName('scoreText');
      if(st) st.setText('Enemies defeated: ' + ( (st.text.match(/\d+/)||[0])[0]*1 + 1 ));
      player.setVelocityY(-200);
    } else {
      // take damage (brief invul)
      if(!player.invul){
        player.health -= 1;
        player.invul = true;
        player.setTint(0xff0000);
        this.time.delayedCall(800, ()=>{ player.invul = false; player.clearTint(); });
        if(player.health <= 0) this.scene.restart({level:1});
      }
    }
  }

  reachGoal(player, goal){
    // trigger a boss encounter scene (simple)
    this.scene.pause();
    const villain = goal.villain;
    const w = this.add.rectangle(this.cameras.main.midPoint.x, 270, 760, 360, 0x05050b, 0.9).setScrollFactor(0);
    const t = this.add.text(this.cameras.main.midPoint.x-300, 150, 'BOSS: ' + villain.name, {font:'28px Arial', fill:'#ff6'}).setScrollFactor(0);
    const desc = this.add.text(this.cameras.main.midPoint.x-300, 200, 'Defeat ' + villain.name + ' to advance!', {font:'18px Arial', fill:'#ddd'}).setScrollFactor(0);
    const btn = this.add.text(this.cameras.main.midPoint.x-30, 320, 'Fight', {font:'20px Arial', fill:'#6f6', backgroundColor:'#222'}).setInteractive().setScrollFactor(0);
    btn.on('pointerdown', ()=>{ this.time.delayedCall(200, ()=>{ this.fightBoss(villain); }); });
    // also allow keyboard attack to start
    this.input.keyboard.once('keydown-J', ()=> this.fightBoss(villain));
  }

  fightBoss(villain){
    // simple boss: spawn an enemy with more HP and a pattern; defeat to go next level
    // clear overlays
    this.children.list.filter(ch=>ch.type==='Text' && ch.text && ch.text.startsWith('BOSS')).forEach(c=>c.destroy());
    this.children.list.filter(ch=>ch.type==='Rectangle' && ch.width===760).forEach(c=>c.destroy());
    this.children.list.filter(ch=>ch.type==='Text' && c).forEach(c=>{});
    // create boss sprite at camera center
    const bx = this.cameras.main.midPoint.x + this.cameras.main.scrollX;
    const boss = this.physics.add.sprite(bx+200, 360, null);
    boss.setSize(80,120);
    boss.health = 3 + Math.floor(this.level/2);
    boss.setTint(0x880088);
    boss.setImmovable(true);
    const g = this.add.graphics();
    g.fillStyle(0x880088,1).fillRect(0,0,80,120);
    g.generateTexture('bossTex'+Math.random(),80,120);
    g.destroy();
    boss.setTexture(Object.keys(this.textures.list).slice(-1)[0]);

    this.physics.add.collider(boss, this.platforms);
    this.physics.add.overlap(this.player, boss, (p,b)=>{
      // if player jumps on boss, damage; else take damage
      if(p.body.velocity.y > 50 && p.y < b.y){
        b.health -= 1;
        p.setVelocityY(-250);
        if(b.health <= 0){
          b.destroy();
          this.levelUp();
        }
      } else {
        if(!p.invul){
          p.health -= 1;
          p.invul = true;
          p.setTint(0xff0000);
          this.time.delayedCall(800, ()=>{ p.invul = false; p.clearTint(); });
          if(p.health <= 0) this.scene.restart({level:1});
        }
      }
    }, null, this);

    // boss attacks periodically (simple projectile)
    this.time.addEvent({
      delay: Math.max(700, 2000 - this.level*50),
      loop: true,
      callback: ()=>{
        if(!boss.scene) return;
        const proj = this.physics.add.image(boss.x-60, boss.y-20, null);
        proj.setSize(14,14);
        const g2 = this.add.graphics();
        g2.fillStyle(0xffff00,1).fillCircle(7,7,7);
        g2.generateTexture('proj'+Math.random(),14,14);
        g2.destroy();
        proj.setTexture(Object.keys(this.textures.list).slice(-1)[0]);
        proj.setVelocityX(-200 - this.level*10);
        proj.setGravityY(200);
        this.physics.add.overlap(this.player, proj, (p,pr)=>{
          pr.destroy();
          if(!p.invul){
            p.health -= 1;
            p.invul = true;
            p.setTint(0xff0000);
            this.time.delayedCall(800, ()=>{ p.invul = false; p.clearTint(); });
            if(p.health <= 0) this.scene.restart({level:1});
          }
        });
      }
    });

  }

  levelUp(){
    if(this.level >= LEVEL_COUNT){
      this.add.text(this.cameras.main.midPoint.x-220, 200, 'CONGRATULATIONS! You defeated Gotham's worst!', {font:'24px Arial', fill:'#6f6'}).setScrollFactor(0);
      return;
    }
    this.level++;
    this.scene.restart({level:this.level});
  }

  update(time, delta){
    if(this.cursors.restart.isDown){
      this.scene.restart({level:this.level});
    }
    if(!this.player) return;
    const left = this.cursors.left.isDown;
    const right = this.cursors.right.isDown;
    const up = this.cursors.up.isDown || this.cursors.w.isDown;
    // movement
    const speed = 180;
    if(left) { this.player.setVelocityX(-speed); }
    else if(right) { this.player.setVelocityX(speed); }
    else { this.player.setVelocityX(0); }

    // jump
    if(up && this.player.body.touching.down){ this.player.setVelocityY(-360); }

    // simple attack: spawn a short-ranged batarang that can damage enemies
    if(Phaser.Input.Keyboard.JustDown(this.cursors.attack)){
      const dir = (this.player.flipX)? -1 : 1;
      const proj = this.physics.add.image(this.player.x + 30*dir, this.player.y, null);
      proj.setSize(12,8);
      const g = this.add.graphics();
      g.fillStyle(0x99ddff,1).fillRect(0,0,12,8);
      g.generateTexture('bat'+Math.random(),12,8);
      g.destroy();
      proj.setTexture(Object.keys(this.textures.list).slice(-1)[0]);
      proj.setVelocityX(400 * (right?1:(left?-1:1)));
      proj.setGravityY(-200);
      this.physics.add.overlap(proj, this.enemies, (p, e)=>{
        p.destroy();
        e.health -= 1;
        if(e.health <= 0) e.destroy();
      });
      this.time.delayedCall(1200, ()=>{ if(proj && proj.destroy) proj.destroy(); });
    }

    // grapple: small teleport to cursor direction (utility)
    if(Phaser.Input.Keyboard.JustDown(this.cursors.grapple)){
      const pointer = this.input.activePointer;
      const tx = pointer.worldX;
      const ty = pointer.worldY;
      const dx = Phaser.Math.Clamp(tx, this.player.x-220, this.player.x+220);
      const dy = Phaser.Math.Clamp(ty, this.player.y-180, this.player.y+30);
      this.player.setPosition(dx, dy);
      this.player.setVelocityY(-80);
    }
  }
}

const config = {
  type: Phaser.AUTO,
  width: 960,
  height: 540,
  parent: 'game-container',
  physics: {
    default: 'arcade',
    arcade: { gravity: { y: 800 }, debug: false }
  },
  scene: [Boot, Game]
};

const game = new Phaser.Game(config);

// Quick keyboard to jump levels for testing (1..9)
window.addEventListener('keydown', (e)=>{
  if(e.key>='1' && e.key<='9'){
    const lvl = parseInt(e.key);
    game.scene.scenes.forEach(s=>{ if(s.scene.key==='Game') s.scene.restart({level:lvl}); });
  }
});
