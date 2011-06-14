Type-Level Programming in Scala, Part 6d: HList Zip/Unzip
-------------------------------------------------------------

Posted by Mark on July 17, 2010

For zip and unzip, we need to define some type classes. First, we will define an HZip type class that accepts two HLists and produces a zipped HList.

.. code-block:: scala

  sealed trait HZip[A <: HList, B <: HList, Result <: HList] {
     def apply(a: A, b: B): Result
  }

We implement the type class with two main cases hzipNil0 and hzipCons.

Zipping two HNils produces an HNil.

.. code-block:: scala

   implicit def hzipNil0 =
      new HZip[HNil, HNil, HNil] {
         def apply(a: HNil, b: HNil) = HNil
      }

Zipping two HCons cells combines their heads into a tuple, zips their tails, and creates a new HCons cell from the tuple and zipped tails.

.. code-block:: scala

   implicit def hzipCons[HA, HB, TA <: HList, TB <: HList, TR <: HList](implicit hzipTail: HZip[TA, TB, TR]) =
      new HZip[HA :: TA, HB :: TB, (HA, HB) :: TR] {
         def apply(a: HA :: TA, b: HB :: TB) = HCons( (a.head, b.head), hzipTail(a.tail, b.tail) )
      }

Two additional cases, hzipNil1 and hzipNil2, handle mismatched lengths by simply truncating the longer list.

.. code-block:: scala

   implicit def hzipNil1[H, T <: HList] =
      new HZip[HCons[H,T], HNil, HNil] {
         def apply(a: HCons[H,T], b: HNil) = HNil
      }

   implicit def hzipNil2[H, T <: HList] =
      new HZip[HNil, HCons[H,T], HNil] {
         def apply(a: HNil, b: HCons[H,T]) = HNil
      }

We hook this into our HListOps type class and we can call zip directly on an HList.

Examples:

.. code-block:: scala

   // a heterogeneous list of length 3 and type
   //  Int :: String :: List[Char] :: HNil
   val a = 3 :: "ai4" :: List('r','H') :: HNil

   // a heterogeneous list of length 4 and type
   //  Char :: Int :: Char :: String :: HNil
   val b = '3' :: 2 :: 'j' :: "sdfh" :: HNil

   // the two HLists zipped
   val c = a zip b

   // zipped again.
   val cc = c zip c.tail

   // verify proper types
   // note that the fourth element of b is dropped, like when zipping a homogeneous List
   val checkCType : (Int, Char) :: (String, Int) :: (List[Char], Char) :: HNil = c

   val checkCCType : ((Int, Char), (String, Int)) :: ((String, Int), (List[Char], Char)) :: HNil = cc

   // verify proper values
   val (3, '3') :: ("ai4", 2) :: (List('r', 'H'), 'j') :: HNil = c

   val ((3,'3'),("ai4",2)) :: (("ai4",2),(List('r', 'H'),'j')) :: HNil = cc

Next, we’ll define an unzip type class that accepts an HList of tuples and separates it into two HLists by components:

.. code-block:: scala

  trait Unzip[H <: HList, R1 <: HList, R2 <: HList] {
     def unzip(h: H): (R1, R2)
  }

Unzipping HNil produces HNil.

.. code-block:: scala

   implicit def unzipNil =
      new Unzip[HNil, HNil, HNil] {
         def unzip(h: HNil) = (HNil, HNil)
      }

For HCons, we unzip the tail, separate the head components, and prepend the respective head component to each tail component.

.. code-block:: scala

   implicit def unzipCons[H1, H2, T <: HList, TR1 <: HList, TR2 <: HList]
      (implicit unzipTail: Unzip[T, TR1, TR2]) =

      new Unzip[(H1,H2) :: T, H1 :: TR1, H2 :: TR2]  {
         def unzip(h: (H1,H2) :: T) = {
            val (t1, t2) = unzipTail.unzip(h.tail)
            (HCons(h.head._1, t1), HCons(h.head._2, t2))
         }
      }

   def unzip[H <: HList, R1 <: HList, R2 <: HList](h: H)(implicit un: Unzip[H, R1, R2]): (R1, R2) =
      un unzip h
  }

Again, we just need to hook this into our HListOps type class.

Building on the example from above,

.. code-block:: scala

   // unzip the zipped HLists
   val (cc1, cc2) = cc.unzip

   val (ca, cb) = cc1.unzip

   // check types
   val checkCC1 : (Int, Char) :: (String, Int) :: HNil = cc1

   val checkCC2 : (String, Int) :: (List[Char], Char) :: HNil = cc2

   val checkCa: Int :: String :: HNil= ca

   val checkCb: Char :: Int :: HNil = cb

We will look at applying functions to HList values next.
