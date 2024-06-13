# Developer Notes

## Sound
For the sound portion of my project, I wanted to simulate a plucked gut string (such as on a viola or violin). You can hear a (very) brief example of a plucked gut string [here](https://youtu.be/ROe61GyvtXM?si=cvGYNr2car3hZIIl&t=79).

I started with the `allolib_playground/tutorials/synthesis/pl-pan.cpp` file and adjusted the settings:
```cpp
createInternalTriggerParameter("amplitude", 0.1, 0.0, 1.0);
createInternalTriggerParameter("frequency", 60, 20, 5000);
createInternalTriggerParameter("attackTime", 0.001, 0.001, 1.0);
createInternalTriggerParameter("releaseTime", 3.0, 0.1, 10.0);
createInternalTriggerParameter("sustain", 0.7, 0.0, 1.0);
createInternalTriggerParameter("Pan1", 0.0, -1.0, 1.0);
createInternalTriggerParameter("Pan2", 0.0, -1.0, 1.0);
createInternalTriggerParameter("PanRise", 0.0, -1.0, 1.0); // range check
```

## Sound - Filter
The first change I made was adding a simple feature to change the sound to a slightly warmer sound:

***Before:***
```cpp
virtual void onProcess(AudioIOData& io) override {
    while(io()){
        mPan.pos(mPanEnv());
        float s1 =  (*this)() * mAmpEnv() * mAmp;
        float s2;
        mEnvFollow(s1);
        mPan(s1, s1,s2);
        io.out(0) += s1;
        io.out(1) += s2;
    }
    if(mAmpEnv.done() && (mEnvFollow.value() < 0.001)) free();
}
```

***After:***
```cpp
float last = 0;
virtual void onProcess(AudioIOData& io) override {
    while(io()){
        mPan.pos(mPanEnv());
        float s1 =  (*this)() * mAmpEnv() * mAmp;
        s1 = (s1 + last) / 2; //FILTER HERE
        last = s1;
        float s2;
        mEnvFollow(s1);
        mPan(s1, s1,s2);
        io.out(0) += s1;
        io.out(1) += s2;
    }
    if(mAmpEnv.done() && (mEnvFollow.value() < 0.001)) free();
}
```

## Sound - Color
Next, I made the most impactful change to the sound—-changing its color. There are 4 different colors: White (default), Brown, Violet, Pink. You can change the color for your simulated sound in the code below:

```cpp
class PluckedString : public SynthVoice {
public:
    float mAmp;
    float mDur;
    float mPanRise;
    gam::Pan<> mPan;
    gam::NoiseWhite<> noise; //CHANGE NOISE COLOR HERE
    gam::Decay<> env;
    gam::MovingAvg<> fil {2};
    gam::Delay<float, gam::ipl::Trunc> delay;
    gam::ADSR<> mAmpEnv;
    gam::EnvFollow<> mEnvFollow;
    gam::Env<2> mPanEnv;
    ...
```

Below are recordings of each sound:

***[White:](https://drive.google.com/file/d/1ZEicDv7yVLLM1MOgnwBqswt5CsxJyGBZ/view?usp=sharing)*** tone is more metallic; a harsher sound like hitting metal strings

***[Brown:](https://drive.google.com/file/d/1xjMj8yc4zmbYa89FWCkVkyrwKycy5ETj/view?usp=sharing)*** warm, wooden-ish sound. Still sounded like hitting strings, and had a 'hidden' kind of sound

***[Violet:](https://drive.google.com/file/d/1rBsYHM9_D3hwTZlhYdhkFRNP0Vj43bwz/view?usp=drive_link)*** more nasal, too metallic and release of note had an electronic/techno tone (more noticeable with different settings)

***[Pink:](https://drive.google.com/file/d/1EdwNBxUo13md_-Spc4m51JO5MRqHZqXG/view?usp=sharing)*** I chose this noise for its scratchy, nasal, wooden tone. The sound is very close to plucking a gut string—-especially with the initial release of the string

## Visualization

In creating a visualization, I wanted to create a larger-scale visualization which would look good in the allolib in 360 degrees.

I started by looking at `allolib_playground/allolib/examples/simulation/waveEquation.cpp` which can be seen below:

https://github.com/allolib-s24/notes-KateUnger/assets/32380357/66e25014-428a-4462-9da2-4111b4e54ddb

However, this was simulating water hitting the ground, rather than water falling. Next I looked at `allolib_playground/allolib/examples/simulation/particleSystem.cpp`

https://github.com/allolib-s24/notes-KateUnger/assets/32380357/48531322-67fe-4ab0-a799-167616fc3788

The particleSystem visualization is broken into two parts: the fountain and the spray. The spray is the small water particles in the bottom left that follow a circle shape and the fountain is the main particle path.

```cpp
template <int M>
void update() {
  for (auto& p : particles) p.update(M);

  for (int i = 0; i < M; ++i) {
    auto& p = particles[tap];

    // fountain
    if (rnd::prob(0.95)) {
      p.vel.set(rnd::uniform(-0.1, -0.05), rnd::uniform(0.12, 0.14),
                rnd::uniform(0.01));
      p.acc.set(0, -0.002, 0);

      // spray
    } else {
      p.vel.set(rnd::uniformS(0.01), rnd::uniformS(0.01),
                rnd::uniformS(0.01));
      p.acc.set(0, 0, 0);
    }
    p.pos.set(4, -2, 0);

    p.age = 0;
    ++tap;
    if (tap >= N) tap = 0;
  }
}
```

I first removed the spray and then moved the particle fountain stream to facing almost straight down (like wind with slight rain). Putting the rain at a slight angle allows the stream to have a fuller volume and look more naturally.

```cpp
template <int M>
void update(float xpos, float ypos) {
  for (auto& p : particles) p.update(M);
  for (int i = 0; i < M; ++i) {
    auto& p = particles[tap];
    // fountain
    if (rnd::prob(0.95)) {
      p.vel.set(rnd::uniform(0.2), 0, 0);
      p.acc.set(0, -0.03, 0);
      if(p.pos[1] < -8) {
        std::printf("pos: %f, \n", p.pos[1]);
      }
    }
    p.pos.set(xpos, ypos, 0);
    p.age = 0;
    ++tap;
    if (tap >= N) tap = 0;
  }
}
```

Next, I created multiple streams of water by adding additional emitters via the three following lines:

```cpp
struct MyApp : public App {
  Emitter<250> em1;
  Emitter<250> em2; // line 1
  ...
```

```cpp
void onAnimate(double dt) {
  em1.update<5>(-4, 0.0);
  em2.update<5>(-3.5, 0.1); // line 2
  ...
  computeParticles(em1);
  computeParticles(em2); // line 3
```

By creating a line of emitters, I was able to create a curtain of rain. I had to put the fountains at different layers in order to make the horizontal lines that can be seen in the video below less emphasized.

https://github.com/allolib-s24/notes-KateUnger/assets/32380357/192a6b11-7d48-4552-a712-9d443f798946

## Final Project

Combined, I played a song live with the computer keyboard on the gut string sound with the rain visualization playing live.

https://github.com/allolib-s24/notes-KateUnger/assets/32380357/f58aa02b-e1a7-42c0-96c1-5faecf0fb296

## Future Ideas
- Bowing gut strings not just plucking
- Adding a better template/class for creating a curtain of emitters, rather than manually adding the three lines for each emitter and manually adjusting their positions
- Create larger rain particles

