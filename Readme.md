 Hexadecachoron encoding of 4D unit vectors


Here I give out an 4d unit vector encoding method that is easy to compute and offers good precision uniformly across the whole range.

Previously I was encoding 3 mesh vertex attributes: normal, tangent and bitangent in the meshlet pages for the hlod as bit streams. Each attribute is a 3D unit vector and two floats should be good enough to parametrize it. The common practice is to use an octhedral encoding as detailed analysed in [1]. 

The major benefit for this approach is that it overcomes the uneven float point distribution to minimize the maximum float quantization error and it is relatively fast to compute. 

However, we could go further take advantage of the fact that, usually normal ,tangent and bitangent are orthogonal to each other. Using them as the rows of 3x3 matrix, we have a matrix of orthogonal-3 group. An orthogonal-3 group is a special orthogonal-3 group plus a sign to show the reflection. The idea is shown in [2]. 

Normalized + Quaternion + sign is exactly a 4D unit vector. I then simply extended the Octahedral normal encoding to the 4D case: 

<ol>
<li>
Problem:
Encode a unit 4D vector {x, y, z, w} that satisfies x*x + y*y + z*z + w * w by three floats, each within -1 and 1.

</li>

<li> 
First step: by L1 normalization, scale the vector by |x|+|y|+|z|+|w|, so it lies on the surface of the 4d Hexadecachoron |x|+|y|+|z|+|w| = 1. This step is homeomorphic.
</li>

<li>
Second step: we want to encoding only 3 floats(3d unit cube). Lets see how to do it in R3. 

If `w >= 0` we just encode x y z from first step, this fills the 3d Octhedral volume`|x| + |y| + |z| < 1`.

Otherwise, `w < 0`, we map it to the outside of the 3d Octhedral volume with in the unit cube:  `|x| + |y| + |z| > 1 && max(|x|, |y|, |z|) < 1`.
</li>

</ol>

The Cpp code:

```
//we should have x*x + y*y + z*z + w*w = 1 and the result outX outY and outZ are all within [-1, 1]

void encode(float x, float y, float z, float w, float& outX, float& outY, float& outZ)
{
    const float l1Norm = abs(x) + abs(y) + abs(z) + abs(w);
    const float l1NormInversed = 1.f / l1Norm;
    outX = x * l1NormInversed;
    outY = y * l1NormInversed;
    outZ = z * l1NormInversed;

    if (w < 0)
    {
        outX = (1.f - abs(outY) - abs(outZ)) * (outX < 0 ? -1.f : 1.f);
        outY = (1.f - abs(outZ) - abs(outX)) * (outY < 0 ? -1.f : 1.f);
        outZ = (1.f - abs(outX) - abs(outY)) * (outZ < 0 ? -1.f : 1.f);
    }
}

//the input x, y, z should be from the encode function above, and we will result in:
//outX*outX + outY*outY + outZ*outZ + outW*outW = 1
void decode(float x, float y, float z, float& outX, float& outY, float& outZ, float& outW)
{
    float w = 1.f - abs(x) - abs(y) - abs(z);
    if(w < 0.f)
        w = w * 0.5f;
    
    const float offset = clamp(-w, 0.f, 1.f);
    x += x > 0.f? -offset : offset;
    y += y > 0.f? -offset : offset;
    z += z > 0.f? -offset : offset;

    const float l = sqrt(x*x + y*y + z*z + w*w);
    outX = x / l;
    outY = y / l;
    outZ = z / l;
    outW = w / l; 
}

```

Discussions:
In practice, the result is quite satisfying, the speed is fast and after comparing using 4-15 bits for each component(x, y and z), 12 bits one component(36 bits total) gives a good rendering result, (visual effect hard to distinguish from normal 3 floats + tangent 3 floats + bitangent 3 floats case).


 `w < 0` range is mapped to a volume 5 times larger than `w > 0` range, but it is in a range with weaker float accuracy. A more quantitive error analysis would be nice using similar approach in [1].

 Note that during the whole process, x is only scaled, and if we want to ensure that the x component carries the sign, we can reapply the sign of float using std::sign_bit and std::copy_sign, which is especially useful for the quaternion + bit encoding case, where plus minus sign makes a big difference. 


---
1. Meyer, Quirin, et al. "On floating‐point normal vectors." Computer Graphics Forum. Vol. 29. No. 4. Oxford, UK: Blackwell Publishing Ltd, 2010.

2. Frey, Ivo Zoltan, and Ivo Herzeg. "Spherical skinning with dual quaternions and QTangents." SIGGRAPH Talks. 2011.
