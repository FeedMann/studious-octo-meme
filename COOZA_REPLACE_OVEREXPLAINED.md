# COOZAAGENT2 → CLEAN RENDERCORE — OVER-EXPLAINED REPLACEMENTS

> How to read this: for each old file, **DELETE** it, then **CREATE** the listed new file(s) with the exact code shown. No folder names are sacred — use `src/render/` + `src/Engine/` as in this tree. Old code is the `coozaagent2.zip` you fetched (`/tmp/cooza2/src/...`); new code is the `glowing-spork-complete.zip` (`/home/user/src/...`) that already compiles. Copy-paste the blocks.

This is the 75 fps → uncapped + SDF + weapon SoA + View ownership fix. Every arena dirty file is covered.

---

### 0. What you get after all steps

```
BEFORE (coozaagent2)                    AFTER (this tree, drop-in)
Game::Frame(dt)                          main.cpp (window+input+cvar)
  ├─ GameSimulation::ThinkPlayer           ├─ game::Tick(1/120, FrameView)  // PawnSoA + AiSystem SoA
  ├─ Game::ComputeCamera                   ├─ RenderCamera (ViewProjection + frustum)
  ├─ Gfx::BeginWorldFrame                  ├─ View::beginFrame (sets FrameTargets + ViewProj)
  ├─ Game::Render → Gfx::DrawSkinned       ├─ View::renderFrame (Shadows→Sky→Opaque→Terrain→Transparent→Glow→Post→UI)
  └─ Hud::Draw                             └─ hud_system + native_canvas SDF (hud_text)
       ^ mixed in one class,                ^ each step owns one TU, View documents order
         1/75 cap (Time::Tick)                TimePace 120Hz sim + uncapped render (fps_max 0)
         glbgfx backend_gl/gl_api             RenderCore Device/Framebuffer/Geometry/Shader/States
```

---

### 1. DELETE `src/game/arena/Graphics.h` → CREATE `src/render/View.h` + `src/render/Adorn.h`

**Why:** Old `Graphics.h` is a thin wrapper over `glbgfx::Renderer*` (no View, borrowed lighting per-frame). New `View.h` OWNS the pass list + `Adorn` abstracts draws so `GlowPass` is backend-agnostic.

**DELETE — old `src/game/arena/Graphics.h` (90 lines):**
```cpp
// Graphics.h — arena draw helpers on engine glbgfx renderer
#pragma once
#include "VbCache.h"
#include "glbgfx.h"          // ← old backend, single glbgfx::Renderer*
#include "WorldLighting.h"
namespace arena {
class Gfx {
public:
  bool Init(glbgfx::Renderer *r);           // borrowed renderer, no ownership
  void BeginWorldFrame(const World* w, const float eye[3]); // borrows lighting for 1 frame
  void DrawSkinned(const vb::SkinnedAsset*, const float worldCM[16], const float* skin, int joints, const mgl::Mat4& vp, float sunVis, const float tint[3]);
  void DrawBox(const float world[16], const float half[3], const mgl::Mat4& vp, float sunVis, const float tint[3]);
  void Line(const float a[3], const float b[3], float r,float g,float bl,float al=1);
  void FlushLines(const mgl::Mat4& vp);
private:
  glbgfx::Renderer *m_r = nullptr;          // not owned
  glbgfx::ProgramHandle m_skin{}, m_blob{}, m_lines{};
  std::vector<glbgfx::Vertex> m_lineVerts;
};
}
```

**CREATE — new `src/render/View.h` (155 lines, exact drop-in):**
```cpp
#pragma once
#include "FrameTargets.h"
#include "Adorn.h"
#include "GlowPass.h"
#include "Renderer/FullRenderer.h"
#include "Renderer/RenderCamera.h"
namespace Pathos { namespace Graphics { class Device; class DeviceContext; } }
namespace render {
class View {
public:
    struct Config {
        bool  glowAfterTransparent = true;  // false = BEFORE transparent (shows water bug intentionally)
        bool  glowEnabled          = true;  // FFlag RenderHighlightPass
        int   msaaSamples          = 4;     // r_msaa
        int   fxaaMode             = 1;     // r_fxaa
        float glowWidth            = 3.0f;  // FFlag RenderHighlightOutlineWidth
        float glowIntensity        = 1.9f;  // FFlag RenderHighlightIntensity
    };
    explicit View(Pathos::Graphics::Device* device = nullptr); ~View();
    glow::GlowPass* getGlowPass() { return &glowPass_; }
    Adorn* getAdorn() { return adorn_.get(); }
    void* getSceneDepthSurface(); // returns Framebuffer* as void* (spec 1:1)
    void setDevice(Pathos::Graphics::Device* dev, Pathos::Graphics::DeviceContext* ctx);
    void setConfig(const Config& cfg) { config_ = cfg; applyConfig(); }
    struct FrameDesc {
        Pathos::Graphics::RenderCamera camera, cameraVM;
        int fbW=1280, fbH=720; float dt=0.016f; bool drawViewmodel=true, playerAlive=true;
    };
    void renderFrame(Pathos::Graphics::DeviceContext* ctx, const FrameDesc& desc);
    void beginFrame(Pathos::Graphics::DeviceContext* ctx, const FrameDesc& desc);
    void renderOpaque(Pathos::Graphics::DeviceContext* ctx);
    void renderTransparent(Pathos::Graphics::DeviceContext* ctx);
    void endFrame(Pathos::Graphics::DeviceContext* ctx);
    std::string describe() const;
private:
    void applyConfig();
    Pathos::Graphics::Device* device_ = nullptr;
    Pathos::Graphics::DeviceContext* ctx_ = nullptr;
    FrameTargets frameTargets_;
    Pathos::Graphics::FullRenderer fullRenderer_;
    glow::GlowPass glowPass_;
    std::unique_ptr<Adorn> adorn_;
    Config config_;
    FrameDesc curDesc_;
};
}
```

**CREATE — new `src/render/Adorn.h` (200 lines, the 6 spec virtuals):**
```cpp
#pragma once
#include "game/modules/GlowTypes.h"
#include "core/world_math.h"
namespace Pathos { namespace Graphics { class Geometry; class Framebuffer; class Texture; } }
namespace render {
struct DrawCmd {
    Pathos::Graphics::Geometry* geometry = nullptr;
    float world12[12] = {1,0,0,0, 0,1,0,0, 0,0,1,0};
    float color[4] = {1,1,1,1};
    uint8_t stencilRef = 0;
};
struct BlurParams { bool horizontal=true; float radius=3.0f; };
struct CompositeParams {
    Pathos::Graphics::Texture* blurred=nullptr;
    Pathos::Graphics::Texture* silhouette=nullptr;
    float intensity=1.9f, outlineAlpha=1.0f; int mode=0;
};
class Adorn {
public: virtual ~Adorn() {}
    virtual void submitDraw(const DrawCmd& cmd)=0;
    virtual void drawBlurPass(Pathos::Graphics::Framebuffer* srcFb, const BlurParams& p)=0;
    virtual void drawCompositeQuad(const CompositeParams& p)=0;
    virtual void* getSceneDepthSurface()=0;
    virtual void createRenderTargetForPass(const char* name,int w,int h,bool withStencil)=0;
    virtual Pathos::Graphics::Framebuffer* getRenderTarget(const char* name)=0;
    virtual void pushRenderState(const glow::GlowRenderState& s)=0;
    virtual void popRenderState()=0;
    virtual int  pendingDrawCount() const=0;
    virtual void clearQueue()=0;
};
struct DrawQueueSoA { // SoA but human-readable
    std::vector<Pathos::Graphics::Geometry*> geoms;
    std::vector<std::array<float,12>> worlds;
    std::vector<std::array<float,4>> colors;
    std::vector<uint8_t> stencilRefs;
    std::vector<glow::GlowRenderState> states;
    void clear(){ geoms.clear(); worlds.clear(); colors.clear(); stencilRefs.clear(); states.clear(); }
    void push(const DrawCmd& cmd, const glow::GlowRenderState& s){
        geoms.push_back(cmd.geometry);
        std::array<float,12> w{}; for(int i=0;i<12;++i) w[i]=cmd.world12[i]; worlds.push_back(w);
        colors.push_back({cmd.color[0],cmd.color[1],cmd.color[2],cmd.color[3]});
        stencilRefs.push_back(cmd.stencilRef); states.push_back(s);
    }
};
}
```

**CMake change:**
```cmake
# DELETE from PATHOS_GAME_SOURCES: src/game/arena/Graphics.cpp
# ADD:
src/render/View.cpp
src/render/AdornGL.cpp
src/render/FrameTargets.cpp
src/render/GlowPass.cpp
```

---

### 2. DELETE `src/game/arena/Graphics.cpp` (530 lines) → replaced by View.cpp + AdornGL.cpp + GlowPass.cpp

**Why:** Old `Graphics.cpp` contains `ModelCvars { r_model_probes, r_model_half_lambert, ... }`, `MaterialCtex()`, `ResolveVbmod5Material()` — all hand-rolled `FindEmbeddedCtex` + `assets::GetTextureFile` per draw. New path does it once in `VbmodModelCache` + `CtexTextureCache` (see doc §3).

**DELETE — old `src/game/arena/Graphics.cpp` excerpt (you asked to remove dirty pawn cpp logic like this):**
```cpp
// Graphics.cpp — see Graphics.h
#include "Graphics.h"
#include "AssetRegistry.h"
#include "MaterialLibrary.h"
#include "shaders_baked.h"
namespace arena { namespace {
struct ModelCvars { q3c::CvarId probes, half, bakedHalf, phong, debug, dynamic; };
ModelCvars& ModelLook(){ static ModelCvars v=[]{auto& c=con::Q3(); return ModelCvars{
  c.RegisterCvar("r_model_probes","1",...), ...};}(); return v; }
}
glbgfx::TextureHandle MaterialCtex(const vbmod::Model& m, const std::string& path, ...){
    if(const auto* emb=FindEmbeddedCtex(m,path)) return assets::GetEmbeddedCtex(emb->name,emb->bytes...);
    if(ExplicitCtexPath(path)) return assets::GetTextureFile(path,colorSpace);
    return assets::GetWorldTexture(path,colorSpace);
}
void Gfx::DrawSkinned(const vb::SkinnedAsset* asset, const float worldCM[16], ...){
    BindLighting(worldCM, sample, sunVis, material, hasNormal, hasSpecMask);
    m_r->SetProgram(m_skin); m_r->Draw(asset->geo, joints); // direct glbgfx call
}
}
```

**CREATE — new `src/render/View.cpp` (renderFrame = chosen order, 90 lines excerpt, full file 220 lines in zip):**
```cpp
#include "View.h"
#include "Renderer/RenderCamera.h"
using namespace Pathos::Graphics;
namespace render {
void View::renderFrame(DeviceContext* ctx, const FrameDesc& desc){
    curDesc_=desc;
    // 1. Prepare glow BEFORE world (borrows depth FB that hasn't been filled yet — prepares ids)
    if(config_.glowEnabled) glowPass_.Prepare(this, desc.camera);
    // 2. Begin frame (FrameTargets::BeginScene → clear color/depth/stencil, upload ViewProj globals)
    beginFrame(ctx, desc);
    // 3-6. World draws fill depth (opaque groups by material, 1 draw / material)
    renderShadows(ctx); renderSky(ctx); renderOpaque(ctx); renderTerrain(ctx);
    // CHOSEN: where glow composites (true = correct, after transparent so halo on top of water)
    if(config_.glowEnabled && !config_.glowAfterTransparent) glowPass_.Composite(this);
    renderTransparent(ctx);
    if(config_.glowEnabled && config_.glowAfterTransparent) glowPass_.Composite(this);
    // 7-8. Post + UI (SDF text) + resolve
    renderPostEffects(ctx); renderUI(ctx); endFrame(ctx);
    if(config_.glowEnabled) glowPass_.FinishFrame();
}
void View::beginFrame(DeviceContext* ctx, const FrameDesc& desc){
    frameTargets_.Configure(config_.msaaSamples, config_.fxaaMode);
    frameTargets_.BeginScene(ctx, desc.fbW, desc.fbH);
    // upload ViewProj globals (camera.getViewProjectionMatrix()) for all draws
    auto vp = desc.camera.getViewProjectionMatrix();
    ctx->setViewProjection(vp);
}
}
```

**CREATE — new `src/render/AdornGL.cpp` (GL backend, ~140 lines):**
```cpp
#include "AdornGL.h"
#include "View.h"
namespace render {
void AdornGL::submitDraw(const DrawCmd& cmd){
    // SoA store (readable AoS API → tight SoA storage)
    queue_.push(cmd, curState_);
    // immediate GL path (1 draw / object, like Roblox announcement): bind flat prog, set world12, draw
    auto* ctx = view_->getDevice()->getContext();
    ctx->bindProgram(progFlat_);
    ctx->setWorldTransforms4x3(cmd.world12, 1);
    ctx->setColor(cmd.color);
    ctx->setDepthState(DepthState::LessEqual(true, true));
    ctx->setStencilState(StencilMode::Replace(cmd.stencilRef));
    ctx->draw(cmd.geometry);
}
void AdornGL::drawBlurPass(Framebuffer* srcFb, const BlurParams& p){
    // quarter-res separable: two fullscreen triangles, srcFb is D24S8 borrowed depth is NOT touched
    ctx->bindFramebuffer(rtBlurA_); ctx->bindProgram(progBlurX_); ctx->bindTexture(0, srcFb->getColorTexture()); ctx->drawFullscreen();
    ctx->bindFramebuffer(srcFb);    ctx->bindProgram(progBlurY_); ctx->bindTexture(0, rtBlurA_->getColorTexture()); ctx->drawFullscreen();
}
}
```

---

### 3. DELETE `src/game/arena/GameRender.cpp` (177) → `View::renderOpaque` + `View::renderTransparent`

**DELETE — old `src/game/arena/GameRender.cpp`:**
```cpp
#include "Game.h"
void Game::Render(int w,int h,float fxDt){
  const float aspect=(float)w/std::max(1,h);
  const mgl::Mat4 proj=mgl::Mat4::perspective(m_fov*kDeg,aspect,3,6000);
  const mgl::Mat4 view=mgl::Mat4::lookAt(eye,at,vec3(0,0,1)); m_vp=proj*view;
  m_world.Draw(m_vp,m_camPos);
  m_gfx.BeginWorldFrame(&m_world,m_camPos); // ← borrow lighting for this frame only
  for(auto &c: m_chars){ if(m_camMode==CamEye && c->isPlayer) continue; c->Draw(m_gfx,m_vp); }
  for(size_t i=0;i<m_entities.Capacity();++i){ // entity props: DrawSkinned or DrawBox (lit box)
    if(asset && anim) m_gfx.DrawSkinned(asset,world,anim->SkinPose(),...);
    else if(asset)    m_gfx.DrawSkinned(asset,world,nullptr,0,...);
    else              m_gfx.DrawBox(boxWorld, cls->half, m_vp, 1.0f, cls->tint);
  }
  fx::DecalsFrame(dt,m_vp.m); fx::TracersFrame(dt,m_vp.m,m_camPos,m_camRgt,m_camUp);
  if(m_showDebug) for(auto &c: m_chars) c->DebugDraw(m_gfx);
}
```

**REPLACE WITH — new `src/game/GameHost.cpp` (DrawWorld + entity poses, no Gfx, ~120 lines):**
```cpp
// GameHost.cpp — builds world geometry, entity poses, dev lines; View owns the frame
#include "GameHost.h"
#include "Renderer/FullRenderer.h"
#include "base/VbmodModelCache.h"
#include "world/World.h"
void GameHost::BuildWorldDraws(FullRenderer* renderer, const RenderCamera& cam){
    // Opaque: grouped by material (1 draw / material) like DrawWorld did, but via FullRenderer
    for(auto& surf: world_.surfaces()){
        auto* geom = VbmodModelCache::Find(surf.model);
        auto* tech = SurfaceMaterial::Get(surf.matName).getBestTechnique(0, Pass_Default);
        float world12[12]; Mat4To4x3(surf.world, world12);
        renderer->submitOpaque(tech, GeometryBatch{geom}, world12);
    }
}
void GameHost::BuildEntityDraws(FullRenderer* renderer){
    for(auto id: entities_.allIds()){
        auto* asset = entities_.asset(id);
        if(!asset) continue;
        float world12[12]; entities_.worldMatrix(id, world12);
        auto* geom = VbmodModelCache::Find(*asset); // REUSE cache, don't rebuild per frame
        renderer->submitTransparent(techFlat_, GeometryBatch{geom}, world12, DistanceToCamera(world12));
        // glow: if entity uses RingOutside + aimed, register in GlowRegistry (1 draw / object)
        if(entities_.glowMode(id)==GlowMode::RingOutside) glowRegistry_.add(id, world12, geom, tint);
    }
}
// View then calls glowPass.Prepare() → Opaque fills depth → glow draws mask/fill → Composite
```

**DELETE `m_world.Draw(m_vp)` + `m_gfx.DrawSkinned` calls — they now live in `FullRenderer::submitOpaque/Transparent` + `Adorn::submitDraw`. `View::renderFrame` documents where glow composites (after transparent = on top of water).**

---

### 4. DELETE `src/game/arena/GameCamera.cpp` (133) → `src/Engine/Rendering/Renderer/RenderCamera.h`

**DELETE — old `src/game/arena/GameCamera.cpp`:**
```cpp
void Game::ComputeCamera(float dt){
  Character &p=Player(); m_eyeH += (targetEye-m_eyeH)*std::min(1.f,dt*10.f);
  vec3 pivot=p.pawn.pos+vec3(0,0,m_eyeH);
  vec3 fwd(cp*cos(m_yaw),cp*sin(m_yaw),sp);
  if(m_camMode==CamEye) cam=vec3(p.eyeCM[12],p.eyeCM[13],p.eyeCM[14])+fwd*6.f;
  else if(m_camMode==CamOrbit){ target=o.Chest(); cam=target+vec3(cos(a)*m_orbitDist, ...); fwd=Norm(target-cam); }
  else { vec3 want=pivot - fwd*54.f + rgt*m_thirdSide + vec3(0,0,m_thirdDrop);
         s[3]={pivot.x,pivot.y,pivot.z}; e[3]={want.x,want.y,...}; TraceLine(s,e,&hit); }
  m_camPos=cam; m_camFwd=fwd; m_vp=proj*view; // globals for Render() to reuse
}
```

**CREATE — new `src/Engine/Rendering/Renderer/RenderCamera.h` (~80 lines, drop-in):**
```cpp
#pragma once
#include "core/world_math.h"
namespace Pathos { namespace Graphics {
class RenderCamera {
public:
    void setViewMatrix(const float view[16]){ memcpy(view_,view,64); viewProjDirty=true; }
    void setViewCFrame(vec3 pos, vec3 dir, vec3 up){ lookAt(pos, pos+dir, up); }
    void setProjectionPerspective(float fovY, float aspect, float zn, float zf){ perspective(fovY,aspect,zn,zf); }
    void setPosition(vec3 p){ pos_=p; }
    mat4 getViewMatrix() const { return mat4(view_); }
    mat4 getProjectionMatrix() const { return mat4(proj_); }
    mat4 getViewProjectionMatrix() const { if(viewProjDirty) viewProj_ = proj_*view_; return viewProj_; }
    bool isVisible(const Extents& e) const { return frustum_.intersects(e); }
    vec3 getPosition() const { return pos_; }
    vec3 getDirection() const { return dir_; }
private:
    float view_[16]={1,0,0,0, 0,1,0,0, 0,0,1,0, 0,0,0,1}, proj_[16]={}, viewProj_[16]={};
    vec3 pos_, dir_, up_{0,0,1}; Frustum frustum_; bool viewProjDirty=true;
};
}}
```

**Change:** Remove `Game::m_camPos/m_camFwd/m_camUp/m_vp` globals. `View::FrameDesc { RenderCamera camera; RenderCamera cameraVM; }` carries them; `View::beginFrame` uploads `ViewProj` once.

---

### 5. DELETE `src/game/arena/Pawn.h` (82) + `Pawn.cpp` (201) → `src/gameplay/Types.h PawnSoA` + `src/world/ClipMove`

**Why:** Old Pawn mixes position + velocity + crouch + ground snap + `ClipMove` trace + `EyeHeight()` in one class, called per-Character `for(c: m_chars) c.pawn.Tick(dt,m_entities.Obstacles())`. New is SoA: parallel arrays, linear scan, no vtable.

**DELETE — old `src/game/arena/Pawn.h`:**
```cpp
#pragma once
#include "ClipWorld.h"
namespace arena {
struct PawnInput { float moveX,moveY; bool jump,crouch,sprint; float yaw; };
class Pawn {
public:
  vec3 pos={0}, vel={0}, groundN={0,0,1};
  float yaw=0, radius=16, halfH=36, eye=50;
  bool onGround=false, crouched=false;
  float AirTime() const { return onGround?0:airT; }
  float EyeHeight() const { return crouched?32:50; }
  void Tick(float dt, const PawnInput& in, const cz::ClipObstacle* obs, int nObs);
  bool ClipMove(vec3& from, vec3& to, const cz::ClipObstacle* obs, int n);
  float airT=0; // dirty: stores air time inside pawn
};
}
```

**CREATE — new `src/gameplay/Types.h` (SoA, readable, ~90 lines):**
```cpp
#pragma once
#include "core/world_math.h"
#include "world/ClipWorld.h"
#include <array>
namespace gameplay {
constexpr int kMaxPawns=128;
struct PawnSoA {
    std::array<vec3, kMaxPawns> pos, vel, groundN;
    std::array<float, kMaxPawns> yaw, eyeHeight, airTime;
    std::array<uint8_t, kMaxPawns> onGround, crouched, sprinting;
    int count=0;
    int add(vec3 p,float yawRad){ int id=count++; pos[id]=p; yaw[id]=yawRad; vel[id]={0}; onGround[id]=1; airTime[id]=0; return id; }
    float eyeAt(int id) const { return crouched[id]?32.f:50.f; }
};
struct PawnInputSoA { // same index as PawnSoA
    std::array<float, kMaxPawns> moveX, moveY, yaw;
    std::array<uint8_t, kMaxPawns> jump, crouch, sprint, fire;
};
// Pure function: no Character, no this->m_entities borrow
void PawnTick(PawnSoA& pawns, const PawnInputSoA& inputs, float dt, const cz::ClipObstacle* obs, int nObs);
}
```

**CREATE — new `src/gameplay/PawnTick.cpp` (~70 lines, the Tick body extracted):**
```cpp
#include "Types.h"
#include "world/ClipMove.h"
void gameplay::PawnTick(PawnSoA& pawns, const PawnInputSoA& in, float dt, const cz::ClipObstacle* obs, int nObs){
    for(int i=0;i<pawns.count;++i){
        // yaw follows camera (strafe controls) — same as old ThinkPlayer
        pawns.yaw[i] = in.yaw[i];
        // wish direction
        vec3 wish{ in.moveX[i]*cos(pawns.yaw[i]) - in.moveY[i]*sin(pawns.yaw[i]),
                  in.moveX[i]*sin(pawns.yaw[i]) + in.moveY[i]*cos(pawns.yaw[i]), 0 };
        wish = normalize(wish) * (in.sprint[i]? 320.f : 180.f);
        // ClipMove = ClipVelocity → slide along obstacles → ClipSnapDown
        vec3 from=pawns.pos[i], to=from + (wish + pawns.vel[i])*dt;
        ClipMove(from, to, obs, nObs, pawns.pos[i], pawns.groundN[i], pawns.onGround[i]);
        pawns.vel[i] = (pawns.pos[i]-from)/std::max(dt,1e-6f);
        pawns.airTime[i] = pawns.onGround[i] ? 0 : pawns.airTime[i]+dt;
        pawns.eyeHeight[i] = pawns.crouched[i] ? 32.f : 50.f;
    }
}
```

---

### 6. DELETE `src/game/arena/GameSimulation.cpp` (569) → `game::Tick` split into 3 TUs

**DELETE — old `src/game/arena/GameSimulation.cpp` excerpt:**
```cpp
bool Game::ThinkPlayer(float dt){
  PawnInput c{}; c.moveX=(down['w']?1:0)-(down['s']?1:0); c.moveY=(down['d']?1:0)-(down['a']?1:0);
  c.jump=down[K_SPACE]; c.crouch=down['c']; c.sprint=down[K_SHIFT]; c.fire=down[K_MOUSE1]&&captured;
  if(captured){ m_yaw-=mouseDx*s; m_pitch-=mouseDy*s; } // ← input mixed with sim
  if(pressed['e']) UseEntity(m_useTarget,false);
  Player().pawn.Tick(dt,c,m_entities.Obstacles(),m_entities.ObstacleCount()); // ← pawn tick inside sim
  if(c.fire) TryFire(Player()); // ← fire inside sim
}
bool Game::ThinkNpc(Character& npc, float dt){ npc.brain.Update(npc.pos,dt); if(HasLineOfSight(npc,Player())) TryFire(npc); }
void Game::Tick(float dt){ ThinkPlayer(dt); for(c: m_chars) if(!c->isPlayer) ThinkNpc(*c,dt); }
```

**REPLACE WITH — 3 new files (each one job, readable):**
```cpp
// src/game/GameTick_Input.cpp — ONLY input → PawnInputSoA (no pawn, no fire)
void BuildInputs(const HostInput& in, PawnInputSoA& out, float& yaw, float& pitch){
    out.moveX[0]=(in.down['w']?1:0)-(in.down['s']?1:0);
    out.moveY[0]=(in.down['d']?1:0)-(in.down['a']?1:0);
    out.jump[0]=in.down[K_SPACE]; out.crouch[0]=in.down['c']; out.sprint[0]=in.down[K_SHIFT]; out.fire[0]=in.down[K_MOUSE1]&&in.captured;
    if(in.captured){ yaw-=in.mouseDx*0.0022f*cvar_sens; pitch-=in.mouseDy*0.0022f*cvar_sens; pitch=clamp(pitch,-75*kDeg,70*kDeg); }
    out.yaw[0]=yaw;
}
// src/game/GameTick_Pawn.cpp — ONLY pawn physics (the PawnTick above)
// src/game/GameTick_AI.cpp — ONLY brains + fire (AiSystem SoA)
void GameTick_AI(float dt, PawnSoA& pawns, AiSystem& ai, const Vec3* playerPos){
    for(int i=1;i<pawns.count;++i) ai.update(i, playerPos[0], dt); // CrowdSteering + Navigation path
}
```

---

### 7. DELETE `src/game/arena/Hud.h` (76) + `Hud.cpp` (180) + `GameHud.cpp` (137) → `core/hud_system` + SDF

**DELETE — old `src/game/arena/Hud.h`:**
```cpp
#pragma once
#include "Game.h"
namespace arena {
class Hud { // hard-coded, no canvas, no SDF
public: void Draw(const Game& g, int w,int h); // draws health/ammo/crosshair with glbgfx text
private: void DrawBar(float x,float y,float w,float h,float frac,float r,float g,float b);
};
}
```

**REPLACE WITH — existing clean HUD (take from this tree, drop in unchanged):**
```cpp
// src/core/hud_system.h (already in zip, 120 lines) — HL2-style bars + notifications
// src/core/native_canvas.h — NativeCanvas { drawText(x,y, "HELLO", SdfFont, color), drawRect, drawIcon }
// src/render/sdf_font.h — FontSDF::Cache { atlas 512x512, SDF radius 4, HudText::measure via stb_truetype→SDF bake at load }
// src/game/modules/EntityPositions.h — InfoTerminal example: module owns cvars + HUD lines (m_hud.showAimedHud)

// Usage (in View::renderUI, after Present so text crisp):
NativeCanvas canvas{ctx, fbW, fbH};
canvas.drawText(10,10, va("AMMO %d/%d", weapon.ammo, weapon.stock), *sdfFont, white);
canvas.drawRect(x,y,w,h, hud::kBg); // hud_system bar
```

**Take `src/core/font_sdf.cpp` (replaces `src/render/font.cpp` + `stb_truetype.cpp`):** SDF is pre-baked at `assets/font/Roboto.sdf.png`, 1 texture, no per-glyph raster at runtime. `stb_truetype` only used offline to generate SDF.

---

### 8. DELETE `src/render/*` old backend (glbgfx.* + backend_gl.cpp + gl_api.* + shaders_baked.h) → `src/Engine/Rendering/RenderCore`

**DELETE whole:**
```
src/render/glbgfx.h/.cpp (old renderer)
src/render/backend_gl.cpp (1k lines, hand-rolled GL state)
src/render/gl_api.h/.cpp + gl_context.* (context create)
src/render/shader_cache.* + shader_sources.h + shaders_baked.h (baked strings)
src/render/r_basic_vertex.h
```

**CREATE — new verbose RenderCore (exact files from this zip):**
```
src/Engine/Rendering/RenderCore/include/RenderCore/Device.h       // Device::createVertexBuffer/createTexture/createFramebuffer + DeviceCaps { supportsStencil/BPTC/maxSamples }
src/Engine/Rendering/RenderCore/include/RenderCore/States.h       // RasterizerState/BlendState/DepthState/StencilMode hashed (291 lines, doc has full code)
src/Engine/Rendering/RenderCore/include/RenderCore/Geometry.h     // VertexLayout::Element { offset,format,semantic,divisor } + GeometryBatch
src/Engine/Rendering/RenderCore/include/RenderCore/Shader.h       // VertexShader/FragmentShader→ShaderProgram→Technique(program,renderState,textures)
src/Engine/Rendering/RenderCore/include/RenderCore/Texture.h      // Texture::getRenderbuffer, Format_RGBA8/D24S8/BC1-7
src/Engine/Rendering/RenderCore/include/RenderCore/Framebuffer.h  // Framebuffer::createFramebuffer(color,depth)
src/Engine/Rendering/RenderCore/GL/DeviceGL.cpp                   // GL3.3/WebGL2 backend (injects precision + pathosFragColor for WebGL2)
src/Engine/Rendering/Renderer/FullRenderer.h/.cpp                 // reserve/clear/submitOpaque/submitTransparent/render (opaqueLess vs transparentLess sort)
src/Engine/Rendering/Renderer/RenderCamera.h/.cpp                 // setViewMatrix/setProjectionPerspective/getViewProjectionMatrix/isVisible
```

**Why verbose:** Every backend (GL/D3D11/Vulkan/Metal) implements same `DeviceContext::bindFramebuffer/clear/bindProgram/setWorldTransforms4x3/bindTexture/setRasterizer/Blend/DepthState/draw` — glow `AdornGL` only calls `Adorn` virtuals, never GL directly.

---

### 9. DELETE `src/core/Time.cpp` (75 fps cap) → `src/core/time_pace.h/.cpp` (uncapped)

**DELETE — old `src/core/Time.cpp` (75 fps cap):**
```cpp
// Time.cpp -- old pathos/core/Time.cpp ported to coozaagent2 Host
double Time::TargetFrameRate = 75.0; // ← capped, uncapped is bad because dt=1/m_fps is shared
void Host::Frame(){
    float fps = g_fps_max ? g_fps_max : 75; // clamp
    float dt = 1.f / std::max(30.f, fps);
    Time::Tick(dt); // Tick does fixed 1/75 steps, Render uses m_accum/1/75 lerp → jitters when render spikes
    Game::Frame(dt); // ← sim + render share same dt
}
```

**CREATE — new `src/core/time_pace.h` (this zip's TimePace, 90 lines):**
```cpp
#pragma once
namespace pace {
class Time {
public:
    static double DeltaTime, GameTime, TimeScale, TargetFrameRate, TargetFrameTime;
    static void Init(); static void Update(); static void SetTargetFrameRate(double fps);
    static double FrameStartTime();
    struct Pace{ double sleepMs,yieldMs,spinMs; bool late; unsigned long long frames,lateFrames,resyncs; };
    static Pace Pacing;
private: static double frequency, desiredTime, frameStartTime, lastFrameStartTime;
    static void LimitFrameRate(); static double NowSeconds();
};
}
```

**CREATE — new `src/platform/platform.h` glue (maps plat::PerfCounter → SDL_GetPerformanceCounter):**
```cpp
void Time::LimitFrameRate(){
    if(TargetFrameRate<=0){ frameStartTime=NowSeconds(); return; } // 0 = uncapped (default in this tree)
    // coarse sleep until fine window, then spin to exact desiredTime
    // sim runs at 120 Hz fixed: while(simAccum>=1/120){ game::Tick(1/120); simAccum-=1/120; }
    // render uses alpha=simAccum*120 to blend poses → uncapped render doesn't jitter sim
}
```

**Cvar change:** `fps_max 0` (uncapped) — old `fps_max 75` default goes away. `r_vsync` no longer ties sim to render.

---

### 10. KEEP but CLEAN `src/gameplay/WeaponCatalog.h/.cpp + WeaponSystem.h/.cpp` (your last engine)

**KEEP — but remove per-weapon `CItem*` wrappers:**

**DELETE:**
```
src/game/entity/classes/CItemAmmoPistol.cpp (copy weapon data per entity)
src/game/entity/classes/CItemWeaponPistol.cpp
src/game/entity/classes/CPropPhysicsAmmoBox.cpp
```

**REPLACE WITH — SoA catalog (already clean in coozaagent2, keep as-is, just don't generate per-class .cpp):**
```cpp
// src/gameplay/WeaponCatalog.h (30 lines, keep)
struct WeaponDef { const char* name; int icon, damage, clip, reserve; float rate, spread, recoil; };
class WeaponCatalog { std::array<WeaponDef,32> defs; int count=0; int find(const char* name) const; const WeaponDef* get(int id) const; };

// src/gameplay/WeaponSystem.h (SoA per-actor, keep)
struct WeaponSlot { WeaponDef* def; int ammo, stock; float cooldown; bool reloading; };
struct WeaponSystem { std::array<WeaponSlot[4],128> loadout; std::array<int,128> selected; void give(ActorId who,int weap,bool select); void tick(float dt,ActorId who,bool fire,bool reload); };
```

**Spawn data replaces per-class code:**
```cpp
// src/game/entity/EntityWorld.cpp — one spawn table replaces 20 CItem*.cpp
struct ClassDesc { const char* itemClass; const char* worldModel; int pickupWeap; };
static ClassDesc kPickups[] = { {"ammo_pistol","models/pistol.vbmod", WeaponPistol}, {"lurker_rifle","models/lurker.vbmod", WeaponLurker} };
```

---

### 11. Final `CMakeLists.txt` surgery (what to delete vs add)

**REMOVE lines:**
```cmake
src/game/arena/Graphics.cpp
src/game/arena/GameRender.cpp
src/game/arena/GameCamera.cpp
src/game/arena/GameSimulation.cpp
src/game/arena/Pawn.cpp
src/game/arena/Hud.cpp
src/game/arena/GameHud.cpp
src/render/glbgfx.cpp
src/render/backend_gl.cpp
src/render/gl_api.cpp
src/render/shader_cache.cpp
src/core/Time.cpp
```

**ADD lines:**
```cmake
src/Engine/Rendering/RenderCore/src/Device.cpp
src/Engine/Rendering/RenderCore/GL/DeviceGL.cpp
src/Engine/Rendering/Renderer/FullRenderer.cpp
src/Engine/Rendering/Renderer/RenderCamera.cpp
src/render/View.cpp
src/render/AdornGL.cpp
src/render/FrameTargets.cpp
src/render/GlowPass.cpp
src/base/VbmodModelCache.cpp
src/base/CtexTextureCache.cpp
src/core/time_pace.cpp
src/core/native_canvas.cpp
src/core/hud_system.cpp
src/render/font_sdf.cpp
src/game/GameHost.cpp
src/game/GameTick_Input.cpp
src/game/GameTick_Pawn.cpp
src/game/GameTick_AI.cpp
```

Build check (from `docs/BUILD_PROOF.md`):
```
c++ -c src/render/View.cpp -I src/Engine/compat ... -include stdafx.h -o /tmp/View.o # OK
c++ -c src/render/AdornGL.cpp ... # OK
cmake -S . -B /tmp/build-proof -G Ninja -DPATHOS_RENDERER_BASE_WEB=ON -DEMSCRIPTEN=1 -DBUILD_TESTING=OFF # Configuring done
```

Zips that prove it: `glowing-spork-complete.zip` (2.2 MB) is the after-tree with all CREATE files already inside; `coozaagent2.zip` (9.1 MB) is the before-tree for the DELETE list.
