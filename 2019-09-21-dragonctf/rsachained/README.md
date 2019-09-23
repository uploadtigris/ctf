# RSA Chained (crypto, 312p, 21 solved)

In the challenge we get [source code](task.py) and [results](output.txt).

The code generates 4 distinctive multiprime RSA keys and encrypts the flag via series of encryptions done with those keys.

For each of the keys we know public key and `d % 2^1050`, so we have a problem known as `partial private key exposure`.
The idea for solution is very similar as in https://github.com/p4-team/ctf/tree/master/2016-09-16-csaw/still_broken_box with the exception that we actuall know much more of the private key than is needed.
The bound is `N/4` exposed bits and we know `N/2`, so it definitely is solvable.

The twist in this task is that we have multiprime RSA and keys are all genreated differently, so we need to do some math on paper to get the right equations.

## General math basis

The general idea for the solution is to use the private key equation for RSA:

```
e * d == 1 mod phi(N)
```

This obviously holds, because it's the basis for RSA decryption, and also we simply know that `d == modinv(e,phi(N))`
We can rewrite the above equation as:

```
e * d - k*phi(N) == 1
```
Directly from the definition of modular equation.

The value `k` is `some integer`, however we know that `d < phi(N)` (from definition, `d` is after all calculated `mod phi(N)`) and both `d` and `e` are positive integers and so is `phi(N)` so `k` can't be more than `e`.
Since `e = 1667` we can brute-force all possible values of `k` so following from here on we will assume `k` is a known constant, and in code we will simply loop over them.

What we know in the task is `d0 = d mod 2**1050`.
Now we change `d` for `d0` getting:

```
e * d0 - k*phi(N) == 1 mod 2**1050
```

This is pretty much what we start with for each of the public keys we have.

## Case N = p*q*r

First case we will tackle is `N = p*q*r` simply because it's the easiest one to spot.
For the other two equations we actually don't know which one is which.

In order to spot for this case, we simply need to calculate GCD of all public keys.
Since there are 2 keys which share factor `r` their GCD will be `r`, while all others will be `1`:

```python
def main():
    n = [
        859120656206379031921714646885063910105407651892527052187347867316222596884434658659405822448255608155810241715879240998190431705853390924506125009673776539735775578734295308956647438104082568455604755897347443474837969627723792555180605570061543466558710984713039896691188382997755705425087392982168984185229388761020298907843938651747237132404355197739247655148879984476412365078220648181358131779226613410257975180399575865809381114503102493767755197618305439592514756307937482535824886677028869214462143121897901687818277055753103973809215799348093165770108731899625440232334370794010877047571087634088643565878666814823597,
        1311485515090222718982495198730831073955174624382380405255882886012541076751694096664143914783345786808989030195914045177845164364400711539537456739170589346033729436625658871146633553503774866142568716068953663977222002088477902235884717082069070796901652820969777833174034685816622756524282480580399883160755030115364291412718061758142515305389808681261201028647918631486855998332674824264495109641835596608336454102370944531225289276734441189842117114187272485368222698210402692987316946307402124818683662256285425749745044473836534317911693431019535047343774304237969249813708212575639444584092633692220540287380100087406907,
        1575060449430659140207638391055943675863526369065063706350412723523667837574603957873052861827978497667790320751709825539894814309966419985565518167069012257059970719629265514554227032833047486506557616694792185308331642271153817395624694602567048186971822198162003259057599067679515651509006583655734337286195372119659961892695887527649792831639865587192165588971284597107150903552624259996427357055727777299373593229142742726141314990452184229476917038184267241036918602554417997784006066104066098902608890274713296413328177121301222893743767266080583504425094092518398681307182308611729107396812972850405735757668088697847951,
        4232819155839550922279592842719433946891627776859962079814516253452165389036653289438928378562503361802962808867376036446065199400114343981489770467719433842467863790025157645790790546711434342749173114584205175937908175583479179580810260063208858154629604787679080148158778144242635384249890271882097552355559004259916015878919322278402739861284711967004042252592561170311676956442870143264815298550428890342085615270647716168020441562257255689477157387477157201225997667848750302348483724394538068236632998615714647043723202692215390463632895984121932442861483996529680353143769212467480570412438553808228897441253339977829074399
    ]
    for c in itertools.combinations(n, 2):
        g = gcd(c[0], c[1])
        if g != 1:
            print(c[0], c[1])
            print(g)
```

This way get get to know that we're looking at keys 1 and 2 and `r = 32619972550448885952992763634430245734911201344234854263395196070105784406551510361671421185736962007609664176708457560859146461127127352439294740476944600948487063407599124272043110271125538616418873138407229`

Now we need to modify the initial equation we have to fit this particular case:

```
e * d0 - k*phi(N) == 1 mod 2**1050
```

`e, d0, k` are all constants so let's focus on `phi(N)`.
We know it is (from definition) `phi(N) = (p-1)*(q-1)*(r-1)` and we know that `p*r*q = N`.

We want to express the value of `phi(N)` as some univariate equation, so we need to get rid of either `q` or `p`:

```
(p-1)*(q-1)*(r-1) = 
(p-1)*(qr-q-r+1) = 
pqr-pq-pr+p-qr+q+r-1 = 
N-pq-pr+p-qr+q+r-1 = | *r
Nr-pqr-pr^2+pr-qr^2+qr+r^2-r =
Nr-N-pr^2+pr-qr^2+qr+r^2-r = |*p
Nrp-Np-p^2r^2+p^2r-pqr^2+pqr+pr^2-rp = 
Nrp-Np-p^2r^2+p^2r-Nr+N+pr^2-pr
```

And now we're left with a term with only a single uknown `p`, because `N` and `r` are constants.

Let's plug this back:

```
e*d0*p*r - k*(Nrp-Np-p^2r^2+p^2r-Nr+N+pr^2-pr) == p*r mod 2**1050
```

We solve this via:

```python
def find_p(d0, e, n, start, stop):
    r = 32619972550448885952992763634430245734911201344234854263395196070105784406551510361671421185736962007609664176708457560859146461127127352439294740476944600948487063407599124272043110271125538616418873138407229
    X = var('X')
    for k in xrange(start, stop):
        print("test for", k)
        results = solve_mod([e*d0*r*X - k*(n*r*X -n*X - X*X*r*r + X*X*r - n*r + n +X*r*r - X*r) == X*r], 2^1050)
        for x in results:
            p0 = ZZ(x[0])
            if is_prime(p0) and gcd(n,p0)!=1:
                return p0
```

And with this we recover value `p0 == p mod 2**1050`.
Notice, however that `p` is 700 bits long, and thus `p0 == p` and we don't need to use approach similar to the one shown in https://github.com/p4-team/ctf/tree/master/2016-09-16-csaw/still_broken_box with Coppersmith method to recover full value of `p`.

This way we break 2 of the 4 keys recovering values of `p`:

```
p1 = 90298557884682577669238320760096423994217812898822512514104930945042122418007925771281125855142645396913218673571816112036657123492733042972301983242487835472292994595416656844378721884370309120262139835889657
p2 = 142270506848638924547091203976235495577725242858694711068289574174127601000137457280276860615471044907560710121669055364010408768146949985099404319539891688093875478389341632242096859500255283810703767020918479
```

## Case N = pq**2

Now let's tackle another case using pretty much the same approach.
The only different is the equation since now `phi(N) = (p-1)*(q-1)*q` and `N = pq^2`

```
(p-1)*(q-1)*q = 
(p-1)*(q^2-q) = 
pq^2-pq-q^2+q = 
N-pq-q^2+q = | *q
Nq-pq^2-q^3+q^2 = 
Nq-N-q^3+q^2
```

Notice we multiply here by `q` and not by `p`, simply because `q` appears already in first and in second power, so getting rid of `q` would be much more difficult.
And again we end up with equation depending on a single variable `q`, so we can plug this back in:

```
e*d0*q - k*(Nq-N-q^3+q^2) == q mod 2**1050
```

And we can again proceed with identical sage code, with the exception for the equation part which is now:

```python
results = solve_mod([e*d0*X - k*(n*X-n-X*X*X + X*X) == X], 2^1050)
```

We don't know if key 3 or 4 is this case, but it doesn't matter, we can run this on both and see which one gives a solution.
It turns out to be key number 2 and we recover q:

```
q3 = 267307309343866797026967908679365544381223264502857628608660439661084648014195234872217075156454448820508389018205344581075300847474799458610853350116251989700007053821013120164193801622760845268409925117073227
```

## Case N=pq where q.nbits() == 1400

Now we need to tacle the last equation, proceeding the same way as before.
In this case more classically `phi(N) = (p-1)*(q-1)`:

```
(p-1)*(q-1) = 
pq-p-q+1 = 
N-p-q+1 = |*p
Np-p^2-pq+p =
Np-p^2-N+p
```

And as before we arrive at equation with a single unknown `p`.
We plug it back in:

```
e*d0*p - k*(Np-p^2-N+p) == p mod 2**1050
```

And we can solve it with the same sage code with equation being:

```python
results = solve_mod([e*d0*X - k*(n*X-X*X-n+X) == X], 2^kbits)
```

And from this we recover the last `p`:

```
p4 = 188689169745401648234984799686937623590015544678958930140026860499157441295507274434268349194461155162481283679350641089523071656015001291946438485044113564467435184782104140072331748380561726605546500856968771
```

## Decrypt the flag

Now what is left is to recover all private keys and decrypt the flag:

```python
import gmpy2

from crypto_commons.generic import long_to_bytes
from crypto_commons.rsa.rsa_commons import modinv

e = 1667


def solve():
    r = 32619972550448885952992763634430245734911201344234854263395196070105784406551510361671421185736962007609664176708457560859146461127127352439294740476944600948487063407599124272043110271125538616418873138407229
    assert gmpy2.is_prime(r)
    n1 = 859120656206379031921714646885063910105407651892527052187347867316222596884434658659405822448255608155810241715879240998190431705853390924506125009673776539735775578734295308956647438104082568455604755897347443474837969627723792555180605570061543466558710984713039896691188382997755705425087392982168984185229388761020298907843938651747237132404355197739247655148879984476412365078220648181358131779226613410257975180399575865809381114503102493767755197618305439592514756307937482535824886677028869214462143121897901687818277055753103973809215799348093165770108731899625440232334370794010877047571087634088643565878666814823597
    p1 = 90298557884682577669238320760096423994217812898822512514104930945042122418007925771281125855142645396913218673571816112036657123492733042972301983242487835472292994595416656844378721884370309120262139835889657
    q1 = (n1 / r) / p1
    assert gmpy2.is_prime(q1)
    assert gmpy2.is_prime(p1)
    d1 = modinv(e, (p1 - 1) * (q1 - 1) * (r - 1))

    n2 = 1311485515090222718982495198730831073955174624382380405255882886012541076751694096664143914783345786808989030195914045177845164364400711539537456739170589346033729436625658871146633553503774866142568716068953663977222002088477902235884717082069070796901652820969777833174034685816622756524282480580399883160755030115364291412718061758142515305389808681261201028647918631486855998332674824264495109641835596608336454102370944531225289276734441189842117114187272485368222698210402692987316946307402124818683662256285425749745044473836534317911693431019535047343774304237969249813708212575639444584092633692220540287380100087406907
    p2 = 142270506848638924547091203976235495577725242858694711068289574174127601000137457280276860615471044907560710121669055364010408768146949985099404319539891688093875478389341632242096859500255283810703767020918479
    q2 = (n2 / r) / p2
    assert gmpy2.is_prime(q2)
    assert gmpy2.is_prime(p2)
    d2 = modinv(e, (p2 - 1) * (q2 - 1) * (r - 1))

    q3 = 267307309343866797026967908679365544381223264502857628608660439661084648014195234872217075156454448820508389018205344581075300847474799458610853350116251989700007053821013120164193801622760845268409925117073227
    n3 = 1575060449430659140207638391055943675863526369065063706350412723523667837574603957873052861827978497667790320751709825539894814309966419985565518167069012257059970719629265514554227032833047486506557616694792185308331642271153817395624694602567048186971822198162003259057599067679515651509006583655734337286195372119659961892695887527649792831639865587192165588971284597107150903552624259996427357055727777299373593229142742726141314990452184229476917038184267241036918602554417997784006066104066098902608890274713296413328177121301222893743767266080583504425094092518398681307182308611729107396812972850405735757668088697847951
    p3 = n3 / (q3 * q3)
    assert gmpy2.is_prime(q3)
    assert gmpy2.is_prime(p3)
    d3 = modinv(e, (p3 - 1) * (q3 - 1) * q3)

    p4 = 188689169745401648234984799686937623590015544678958930140026860499157441295507274434268349194461155162481283679350641089523071656015001291946438485044113564467435184782104140072331748380561726605546500856968771
    n4 = 4232819155839550922279592842719433946891627776859962079814516253452165389036653289438928378562503361802962808867376036446065199400114343981489770467719433842467863790025157645790790546711434342749173114584205175937908175583479179580810260063208858154629604787679080148158778144242635384249890271882097552355559004259916015878919322278402739861284711967004042252592561170311676956442870143264815298550428890342085615270647716168020441562257255689477157387477157201225997667848750302348483724394538068236632998615714647043723202692215390463632895984121932442861483996529680353143769212467480570412438553808228897441253339977829074399
    q4 = n4 / p4
    assert gmpy2.is_prime(q4)
    assert gmpy2.is_prime(p4)
    d4 = modinv(e, (p4 - 1) * (q4 - 1))

    ct = 594744523070645240942929359037746826510854567332177011620057998249212031582656570895820012394249671104987340986625186067934908726882826886403853350036347685535238091672944302281583099599474583019751882763474741100766908948169830205008225271404703602995718048181715640523980687208077859421140848814778358928590611556775259065145896624024470165717487152605409627124554333901173541260152787825789663724638794217683229247154941119470880060888700805864373121475407572771283720944279236600821215173142912640154867341243164010769049585665362567363683571268650798207317025536004271505222437026243088918839778445295683434396247524954340356

    rsa1 = (n1, d1)
    rsa2 = (n2, d2)
    rsa3 = (n3, d3)
    rsa4 = (n4, d4)

    rsa = sorted([rsa1, rsa2, rsa3, rsa4], reverse=True)

    for n, d in rsa:
        ct = pow(ct, d, n)
    print(long_to_bytes(ct))


solve()
```

And we get `DrgnS{w3_fiX3d_that_f0r_y0U}`