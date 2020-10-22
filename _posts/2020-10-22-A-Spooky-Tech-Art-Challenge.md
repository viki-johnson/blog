---
layout: post
title: A Spooky Tech Art Challenge 
date: '2020-10-22 00:09:00 +0300'
description: >-

img: 2020-10-22-00.png
tags:
  - unity3d
  - tech art
  - shaders
  - challenges
published: true
---
# A Spooky Tech Art Challenge 


Initially I wasn’t very keen on this fortnight’s theme of “replication” for the Tech Art Challenge. It was a little bit too open, I had an idea of copying the “swimming through swamp water effect” from The Last of Us 2, but then, at work, we were discussing games with different effects and Perception came up.

Perception is a survival horror game, made by The Deep End Games in 2017. And the main mechanic of it is that the heroine, Cassie, is blind. She can only see by echolocation, so she taps her cane and can see where a clock chimes and pipes rumble.

The view the player has is a very ghostly blue vision, adding to the eerie atmosphere of this New England mansion.

{:refdef: style="text-align: center;"}
![ ]({{site.baseurl}}/assets/img/2020-10-22-01.jpg)
{: refdef}

I haven’t actually played Perception, but from watching some videos of the gameplay I had a pretty good idea of how this was done, or rather how I would do it. I had made post processing shaders before that use depth, or distance from a set point to transition between two camera views, so that’s where I started.

## Part One: The Post Processing Shader

I used this tutorial as a basis: [Shaders Case Study - No Man’s Sky: Topographic Scanner - YouTube](https://www.youtube.com/watch?v=OKoNp2RqE9A). In which a scanner moves out from a specific point. The placement of the scan line is based on the “scan distance” from the set point, which gradually increases.

But I don’t need the whole of this shader and script. My properties look like this:

~~~
    Properties
    {
        _Black(“Black”, Color) = (1, 1, 1, 0)
        _Edge(“Edge Softness”, float) = 0
    }
~~~

Black is the dark colour, it shouldn’t be completely #000000, but pretty dark. I went for a rather dark blue. And the edge variable controls how soft the edges of the light circles are.

Next is where the properties are declared:
~~~
            sampler2D _MainTex, _CameraDepthTexture;

            float4 _WorldSpaceScannerPos[100];
            float _ScanDistanceArr[100];
            int _ArrayLength;

            float _Black, _Edge;
~~~

We need a lot less than in the original, but we are adding two arrays and an integer to track the array. See the tutorial uses one point of origin for the scan, only one point to take information from and we need more. Maybe 100 is overkill. But we can’t add to these arrays once they’re set.

Now the fragment shader:


~~~
            half4 frag (VertOut i) : SV_Target
            {

                half4 col = tex2D(_MainTex, i.uv);

                float rawDepth = DecodeFloatRG(tex2D(_CameraDepthTexture, i.uv_depth));
                float linearDepth = Linear01Depth(rawDepth);
                float4 wsDir = linearDepth * i.interpolatedRay;
                float3 wsPos = _WorldSpaceCameraPos + wsDir;
                half4 scannerCol = _Black;

                float mask = 0;

                for(int I=0; I<_ArrayLength; I++)
                {
                    float3 dist = distance(wsPos, _WorldSpaceScannerPos[I]);
                    float3 sphere = 1 - saturate(dist/_ScanDistanceArr[i]);
                    sphere = saturate(sphere* _Edge);

                    mask += sphere.r;
                }

                mask = saturate(mask);

                return lerp(scannerCol, col, mask);
            }

~~~

The top chunk of this is pretty much the same, but we set scannerCol as the dark colour we selected earlier. 

The script inside the for loop returns what is essentially a blob around each point in the _WorldSpaceScannerPos and adds it to my mask.  Note that this array doesn’t use the actual length of either array, rather the ArrayLength int.

I might overuse saturate, but it ensures the value stays between 0 and 1, and that we don’t get any crazy oversaturated bright parts.

Then at end we return a lerp between the camera feed and the dark colour, based on the mask we just made.

## Part Two: The Script

Again large parts of this are the same. But we need to add some more properties:

~~~
    public Transform character;
    public Material EffectMaterial;
    public float growSpeed, shrinkSpeed, ScanMax;
    private Camera _camera;
    public int nodeCount = 0;
    public bool growing;
    public Vector4[] posArray = new Vector4[100];
    public float[] distArray = new float[100];
~~~

We’re going to change what happens in update:

~~~
    void Update()
    {
        if (Input.GetKeyDown(KeyCode.C))
        {
            growing = true;
            posArray[nodeCount] = character.position;
            nodeCount ++;
            EffectMaterial.SetFloat(“_ArrayLength”, nodeCount);

            StartCoroutine(Scan());
        }
    }
~~~

We’re keeping GetKeyDown in update and moving the scanning to a Coroutine.  We just want to put our current position in the array, and add one to the nodeCount, which gets passed straight to the shader (remember, we need this so that we don’t have to run through alllll the entries in the array each time).

~~~
    IEnumerator Scan()
    {
        while(distArray[nodeCount-1] < ScanMax)
        {
            distArray[nodeCount-1] += Time.deltaTime * growSpeed;
            yield return null;
        }

        yield return new WaitForSeconds(0.1f);

        growing = false;
        while(!growing)
        {
        for(int I = 0; I<nodeCount; I++)
        {
            if(distArray[I] > 0)
            {
                distArray[I] -= Time.deltaTime * shrinkSpeed;
                distArray[i] = Mathf.Max(distArray[i], 0);
            }
            yield return null;
        }
        }
    }
~~~

So in this coroutine begins by growing out a circle of light around the user, if the light just pops into place, then it can be a little jarring. Then we cycle through all the other nodes and begin to shrink each of them, slowly. Because each node has its own scanDistance, instead of using just one, we can control each individually.

Finally we edit the OnRenderImage:

~~~
    [ImageEffectOpaque]
    void OnRenderImage(RenderTexture src, RenderTexture dst)
    {
        EffectMaterial.SetVectorArray(“_WorldSpaceScannerPos”, posArray);
        EffectMaterial.SetFloatArray(“_ScanDistanceArr”, distArray);
        RaycastCornerBlit(src, dst, EffectMaterial);
    }
~~~

We just need to set both of the arrays in the shader.  And that’s it! More or less. But we want some of that nice blue ghostliness.

## Part Three: Fresnal Shader

For this, I wanted to use a fresnel shader. And grabbed a nice basic one from Ronja](https://www.ronja-tutorials.com/2018/05/26/fresnel.html), who writes amazing shader tutorials.

I did away with the albedo, because the emission is all I care about. And so repurposed the Main Tex as a noise, which adds extra spooky to the scene.

~~~
            float2 uv = i.uv_MainTex;

            fixed4 noise0 = tex2D(_MainTex, uv);
            fixed4 noise1 = tex2D(_MainTex, uv / 3 + _Time.y * _Speed);
            fixed4 noise2 = tex2D(_MainTex, uv / 3 - _Time.y * _Speed);
            fixed4 noise = saturate((noise0 + noise1 + noise2)*_Intensity);

            float fresnel = dot(i.worldNormal*noise.r, i.viewDir);
~~~

That’s added to the fresnel, and left the rest the same.


{:refdef: style="text-align: center;"}
![ ]({{site.baseurl}}/assets/img/2020-10-22-02.gif)
{: refdef}

The model I used is a [free house off of TurboSquid](https://www.turbosquid.com/3d-models/3d-house-1628048). I wanted something with lots of edges to show the shaders. But it would have looked a lot better if I'd used models made specifically for this, then I could have used different noise textures for different things to show wood or fabrics or something.

But altogether I'm pretty pleased with how it turned out. Luckily my first worked really well, so I didn't have to spend too much time experimenting, like the watercolour one. 